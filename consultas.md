# CONSULTAS

## 1. Devolver el nombre y la descripción de aquellos objetos que no se pueden crear mediante una receta ni se pueden obtener de un monstruo y se utilizan para craftear o mejorar algún arma. 
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

```cypher
MATCH (r:Recipe)-[prod:PRODUCES]->(output:Item)
MATCH (r)-[:NEEDS]->(input:Item)

WITH r, output, prod.amount AS outputAmount,
     collect(input) AS inputs,
     sum(input.value) AS totalInputValue

// ALL() verifica que CADA item input NO sea producido por ninguna Recipe
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
