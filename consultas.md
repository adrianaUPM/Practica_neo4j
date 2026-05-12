# CONSULTAS

## 1. Devolver el nombre y la descripción de aquellos objetos que no se pueden crear mediante una receta ni se pueden obtener de un monstruo y se utilizan para craftear o mejorar algún arma. 

Buscamos items que cumplan simultáneamente tres condiciones:
- NO existe ninguna Recipe que los produzca (NOT EXISTS)
- NO los dropea ningún Monster (NOT EXISTS)
- SÍ son necesarios para algún WeaponCraft o WeaponUpgrade (EXISTS)

Usamos subqueries existenciales `(EXISTS { MATCH ... }) `que son más eficientes que `WHERE NOT (i)<--()` porque no generan resultados intermedios.

```cypher
MATCH (i:Item)
WHERE NOT EXISTS { MATCH (:Recipe)-[:PRODUCES]->(i) }
  AND NOT EXISTS { MATCH (:Monster)-[:DROPS]->(i) }
  AND (
    EXISTS { MATCH (:WeaponCraft)-[:NEEDS]->(i) }
    OR
    EXISTS { MATCH (:WeaponUpgrade)-[:NEEDS]->(i) }
  )
RETURN i.name AS nombre, i.description AS descripcion
ORDER BY i.name;
```

## 2. El id de receta, nombre de los items usados como input y nombre del item generado como output de las 5 recetas que más ganancias generan. La ganancia que genera una receta es el valor total de los items que genera, menos el coste total de los items que necesita. Todos los items que se usan en la receta no deben ser producidos por otras recetas.1* 
1* Revisa el funcionamiento de la funcion ALL para forzar que todos los elemntos de una lista cumplan una 
condición https://neo4j.com/docs/cypher-manual/current/functions/predicate/ 

Ganancia = (valor del output × cantidad) − suma(valor de cada input). La restricción: todos los items de input NO deben ser producidos por otra Recipe. Usamos `ALL(x IN lista WHERE condición)` para verificar que el predicado se cumple para cada elemento de la lista.

Primero recopilamos todos los inputs de cada receta en una lista, luego verificamos que ninguno de ellos aparezca como output de otra Recipe.

```cypher
MATCH (r:Recipe)-[prod:PRODUCES]->(output:Item)
MATCH (r)-[:NEEDS]->(input:Item)

WITH r, output, prod.amount AS outputAmount,
     collect(input) AS inputs,
     sum(input.value) AS totalInputValue

// ALL() verifica que cada item input NO sea producido por ninguna Recipe
WHERE ALL(inp IN inputs WHERE NOT EXISTS {
    MATCH (:Recipe)-[:PRODUCES]->(inp)
})

WITH r,
     output,
     [inp IN inputs | inp.name] AS inputNames,
     (output.value * outputAmount) - totalInputValue AS ganancia

RETURN r.id AS id_receta,
       inputNames AS items_input,
       output.name AS item_output,
       ganancia
ORDER BY ganancia DESC
LIMIT 5;
```


## 3. El nombre y el tipo del arma que se puede craftear directamente y que más daño hace de tipo fuego. Además, incluir los monstruos que se deben matar para crearla, así como cualquier material extra que se necesario para fabricar el arma y que no se obtiene matando dichos monstruos. 

- Encontrar el arma crafteable `(WeaponCraft-[:PRODUCES]→Weapon)` con mayor daño en `DEALS` donde el Element tiene `name='fire'`.
- Para esa arma, obtener todos los items del WeaponCraft y qué monstruos los dropean.
- Los materiales extra son aquellos items que no dropea ninguno de los monstruos identificados en el anterior punto.
La cosa está en definir correctamente "materiales extra", no como que el item no lo dropee nadie, sino que no lo dropea ninguno de los monstruos que ya debemos matar.

```cypher
// Arma crafteable con más daño de fuego
MATCH (wc:WeaponCraft)-[:PRODUCES]->(w:Weapon)
MATCH (w)-[d:DEALS]->(e:Element)
WHERE toLower(e.name) = 'fire'
WITH w, d.damage AS fire_damage
ORDER BY fire_damage DESC LIMIT 1

// Items necesarios y sus monstruos
MATCH (wc2:WeaponCraft)-[:PRODUCES]->(w)
MATCH (wc2)-[:NEEDS]->(item:Item)
OPTIONAL MATCH (m:Monster)-[:DROPS]->(item)
WITH w, collect(DISTINCT item) AS allItems,
     collect(DISTINCT m) AS allMonsters

//Materiales extra = items no dropeados por allMonsters
WITH w,
     [m IN allMonsters WHERE m IS NOT NULL | m.name] AS monsterNames,
     [i IN allItems WHERE NOT EXISTS {
         MATCH (mon:Monster)-[:DROPS]->(i)
         WHERE mon IN allMonsters
     }] AS extraItems

RETURN w.name AS nombre_arma,
       w.kind AS tipo_arma,
       monsterNames AS monstruos_a_matar,
       [i IN extraItems | i.name] AS materiales_extra;
```

## 4. Por cada tipo de arma, mostrar el nombre del arma o armas de ese tipo que más daño combinado hace, es decir, la suma de su daño base más todos los daños extra que sean de kind “element”. Devolver los tipos ordenados de mayor cantidad a menor cantidad de daño y mostrar el daño que hace dicha arma. 

El daño combinado = `Weapon.damage (base) + suma de los DEALS`.damage donde el Element tiene kind = 'element' (no contamos los estados alterados como poison, sleep, etc.).

Usamos `OPTIONAL MATCH` porque no todas las armas tienen daño elemental. `COALESCE(d.damage, 0) `convierte los NULL en 0. Luego agrupamos por tipo, calculamos el máximo y filtramos las armas que igualan ese máximo (puede haber empates).

```cypher
MATCH (w:Weapon)
OPTIONAL MATCH (w)-[d:DEALS]->(e:Element)
WHERE e.kind = 'element'

WITH w,
     w.damage + sum(COALESCE(d.damage, 0)) AS total_damage

WITH w.kind AS tipo,
     max(total_damage) AS max_damage,
     collect({nombre: w.name, damage: total_damage}) AS armas

WITH tipo, max_damage,
     [a IN armas WHERE a.damage = max_damage | a.nombre] AS mejores_armas

RETURN tipo, mejores_armas AS armas, max_damage AS daño_maximo
ORDER BY max_damage DESC;
```


## 6. El nombre de los monstruos que tengo que matar para fabricar el arma con id 880. Ten en cuenta que dicha arma no se puede craftear directamente, solo se puede obtener mediante actualización de otras armas. 

**consultas auxiliares**

```cypher
MATCH (w:Weapon)
WHERE w.id = 880
RETURN w.name, w.id
```
"Alarido sangriento"

```cypher
// Ver toda la cadena del arma 880 siguiendo TRANSFORMS
MATCH path = (origen)-[:TRANSFORMS*1..20]->(w880:Weapon {id: 880})
RETURN [n IN nodes(path) | {label: labels(n), id: n.id, name: n.name}] AS cadena
```
```json
[
  {
    name: null,
    label: ["WeaponUpgrade"],
    id: 880
  },
  {
    name: "Alarido sangriento",
    label: ["Weapon"],
    id: 880
  }
]
...
```

```cypher
// Ver qué arma base está conectada a ese upgrade
MATCH (wu:WeaponUpgrade)-[:TRANSFORMS]->(w:Weapon {id: 880})
MATCH (wu)-[r]-(n)
RETURN type(r) AS relacion, labels(n) AS vecino, 
       CASE WHEN startNode(r) = wu THEN 'SALIENTE' ELSE 'ENTRANTE' END AS direccion,
       n.id AS id_vecino, n.name AS nombre_vecino
```

**Resultado:**
```cypher
MATCH (w880:Weapon {id: 880})

// La cadena usa solo TRANSFORMS en ambas direcciones
MATCH path = (origen:Weapon)-[:TRANSFORMS*1..20]->(w880)
WHERE EXISTS { MATCH (:WeaponCraft)-[:PRODUCES]->(origen) }

WITH DISTINCT [n IN nodes(path) WHERE 'Weapon' IN labels(n)] AS armas_cadena

UNWIND armas_cadena AS arma

// Los items necesarios están en el WeaponUpgrade conectado por TRANSFORMS
OPTIONAL MATCH (wc:WeaponCraft)-[:PRODUCES]->(arma)
OPTIONAL MATCH (wc)-[:NEEDS]->(ic:Item)
OPTIONAL MATCH (wu:WeaponUpgrade)-[:TRANSFORMS]->(arma)
OPTIONAL MATCH (wu)-[:NEEDS]->(iu:Item)

WITH collect(DISTINCT ic) + collect(DISTINCT iu) AS todos_items

UNWIND todos_items AS item
WITH item WHERE item IS NOT NULL
MATCH (m:Monster)-[:DROPS]->(item)
RETURN DISTINCT m.name AS nombre_monstruo
ORDER BY m.name
```
