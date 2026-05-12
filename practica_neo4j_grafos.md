```python
from neo4j import GraphDatabase

NEO4J_URI      = "bolt://neo4j:7687"
NEO4J_USER     = "neo4j"
NEO4J_PASSWORD = "BDII2023"
NEO4J_DATABASE = "dungeons"

driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))

with driver.session(database=NEO4J_DATABASE) as session:
    result = session.run("MATCH (w:Weapon) RETURN count(w) AS total")
    print("Armas en la base de datos:", result.single()["total"])
```

    Armas en la base de datos: 1024



```python
with driver.session(database=NEO4J_DATABASE) as session:
    result = session.run("""
        MATCH (w:Weapon)
        OPTIONAL MATCH (wc:WeaponCraft)-[:PRODUCES]->(w)
        OPTIONAL MATCH (wc)-[:NEEDS]->(ic:Item)
        OPTIONAL MATCH (wu:WeaponUpgrade)-[:TRANSFORMS]->(w)
        OPTIONAL MATCH (wu)-[:NEEDS]->(iu:Item)
        WITH w, collect(DISTINCT ic.name) + collect(DISTINCT iu.name) AS items
        WHERE size(items) > 0
        RETURN w.name AS arma, items
        LIMIT 5
    """)
    for r in result:
        print(r["arma"], "->", r["items"])
```

    Cañón de esperanza II -> ['Hierro']
    Cañón de esperanza III -> ['Machalita', 'Dragonita']
    Cañón de esperanza IV -> ['Carbalita']
    Cañón de esperanza V -> ['Hueso escudo', 'Fucium']
    Cartuchera Chata I -> ['Escama de Chatacabra', 'Mandíbula de Chatacabra', 'Piel de Chatacabra']



```python
from neo4j import GraphDatabase
from graphdatascience import GraphDataScience
import pandas as pd
pd. set_option('display.max_columns', None)

NEO4J_URI      = "bolt://neo4j:7687"
NEO4J_USER     = "neo4j"
NEO4J_PASSWORD = "BDII2023"
NEO4J_DATABASE = "dungeons"

gds = GraphDataScience(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD), database=NEO4J_DATABASE)
driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))

def run_query(query, parameters=None):
    with driver.session(database=NEO4J_DATABASE) as session:
        result = session.run(query, parameters or {})
        return [record.data() for record in result]

print("GDS version:", gds.version())
print("Conexión OK")
```

    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('advertisedListenAddress' returned by 'gds.debug.arrow' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.debug.arrow()'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('serverLocation' returned by 'gds.debug.arrow' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.debug.arrow()'


    GDS version: 2026.4.0
    Conexión OK



```python
pd.set_option('display.max_colwidth', None)
```

---
## Paso 1: Crear aristas SIMILAR con índice de Jaccards?
Queremos unir dos nodos `Weapon` con una relación `SIMILAR` si comparten al menos un `Item` en sus procesos de fabricacn hay?
Del esquema:
- `WeaponCraft -[:NEEDS]-> Item` (crafteo desde cero)
- `WeaponUpgrade -[:NEEDS]-> Item` (actualización)
- `WeaponCraft -[:PRODUCES]-> Weapon` (el craft produce un arma)
- `Weapon -[:TRANSFORMS]-> WeaponUpgrade` (el upgrade transforma un arma en otra)
- `WeaponUpgrade -[:PRODUCES]-> Weapon` ← (implícito: el upgrade produce el arma destino)

### Índice de Jaccard
El índice de Jaccard entre dos conjuntos A y B es:
$$J(A,B) = \frac{|A \cap B|}{|A \cup B|}$$

Ejemplo del enunciado:
- Arma 1: {Dragonita, Cola de Quematrice, Piedra de fuego} → |A| = 3
- Arma 2: {Cristal de tierra, Machalita, Dragonita}       → |B| = 3
- Intersección: {Dragonita}                               → |A∩B| = 1
- Unión: 3 + 3 - 1 = 5                                   → |A∪B| = 5


### Estrategia Cypher
1. Para cada `Weapon`, recopilar el conjunto de `Item` ids que necesita (via sus WeaponCraft y WeaponUpgrade).
2. Para cada par de armas que comparten al menos un item, calcular Jaccard.
3. Crear/actualizar la relación `SIMILAR > ght calculado.

> **Nota:** Usamos `MERGE` para evitar duplicados si se ejecuta varias veces.


```python
#Limpieza preventiva  borrar relaciones SIMILAR previas
#esto nos asegura idempotencia si ejecutamos la celda más de una vez.
cleanup_query = """
MATCH ()-[s:SIMILAR]->()
DELETE s
"""
run_query(cleanup_query)
print("relaciones SIMILAR previas eliminadas")
```

    relaciones SIMILAR previas eliminadas



```python
#creamos relaciones SIMILAR con peso Jaccard
#- Recopilamos para cada arma los items que necesita
#- Comparamos todos los pares de armas (w1.id < w2.id para evitar duplicados)
#- Calculamos la intersección y unión de sus conjuntos de items
#- Si la intersección >= 1, creamos SIMILAR con peso = |intersección| / |unión|

create_similar_query = """
// Recopilamos los items de cada arma
// Un arma puede tener WeaponCraft y/o WeaponUpgrade asociados
MATCH (w:Weapon)
OPTIONAL MATCH (wc:WeaponCraft)-[:PRODUCES]->(w)
OPTIONAL MATCH (wc)-[:NEEDS]->(ic:Item)
OPTIONAL MATCH (wu:WeaponUpgrade)
WHERE (w)-[:TRANSFORMS]->(wu) OR 
      EXISTS { MATCH (prev:Weapon)-[:TRANSFORMS]->(wu2:WeaponUpgrade)-[:PRODUCES]->(w) WHERE wu = wu2 }
// Nota: la relación TRANSFORMS va de Weapon -> WeaponUpgrade (el arma base)
// y WeaponUpgrade -> PRODUCES -> Weapon (el arma resultado)
// Entonces, para el arma resultado buscamos el upgrade que la produce
WITH w, collect(DISTINCT ic.id) AS craftItems

// Ahora buscamos los items del upgrade que produce esta arma
OPTIONAL MATCH (wu:WeaponUpgrade)-[:PRODUCES]->(w)
OPTIONAL MATCH (wu)-[:NEEDS]->(iu:Item)
WITH w, craftItems, collect(DISTINCT iu.id) AS upgradeItems

// Unimos ambos conjuntos de items
WITH w, [x IN craftItems + upgradeItems WHERE x IS NOT NULL] AS allItems
WHERE size(allItems) > 0

// Guardamos en memoria para comparar pares
WITH collect({weapon: w, items: allItems}) AS weaponItems

// Generamos todos los pares (i < j para no duplicar)
UNWIND range(0, size(weaponItems) - 2) AS i
UNWIND range(i + 1, size(weaponItems) - 1) AS j
WITH weaponItems[i] AS a, weaponItems[j] AS b

// Calculamos intersección y unión
WITH a.weapon AS wa, b.weapon AS wb,
     [x IN a.items WHERE x IN b.items] AS intersection,
     apoc.coll.union(a.items, b.items) AS union_items

// Solo creamos arista si hay intersección
WHERE size(intersection) > 0

WITH wa, wb,
     toFloat(size(intersection)) / toFloat(size(union_items)) AS jaccard

// Creamos la relación bidireccional (una en cada dirección para facilitar GDS)
MERGE (wa)-[s:SIMILAR]-(wb)
SET s.weight = jaccard

RETURN count(*) AS relaciones_creadas
"""

result = run_query(create_similar_query)
print(f"Relaciones SIMILAR creadas: {result[0]['relaciones_creadas']}")
```

    Relaciones SIMILAR creadas: 575



```python
verify_query = """
MATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)
WHERE id(w1) < id(w2)
RETURN w1.name AS arma1, w2.name AS arma2, round(s.weight * 100) / 100 AS jaccard
ORDER BY jaccard DESC
LIMIT 10
"""
df = pd.DataFrame(run_query(verify_query))
print("Top 10 pares de armas más similares:")
display(df)
```

    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 7, offset: 49} for query: '\nMATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)\nWHERE id(w1) < id(w2)\nRETURN w1.name AS arma1, w2.name AS arma2, round(s.weight * 100) / 100 AS jaccard\nORDER BY jaccard DESC\nLIMIT 10\n'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 16, offset: 58} for query: '\nMATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)\nWHERE id(w1) < id(w2)\nRETURN w1.name AS arma1, w2.name AS arma2, round(s.weight * 100) / 100 AS jaccard\nORDER BY jaccard DESC\nLIMIT 10\n'


    Top 10 pares de armas más similares:



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>arma1</th>
      <th>arma2</th>
      <th>jaccard</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Punzadora potente II</td>
      <td>Arco poder salvaje II</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Cartuchera Chata III</td>
      <td>Segur Chata III</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Arco huracán II</td>
      <td>Garra vendaval II</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Arco de cazador I</td>
      <td>Aspa ósea I</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Ira Reina I</td>
      <td>Caro Lutemis I</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Zoh Samira I</td>
      <td>Zoh Yirmiya I</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Lluvia bendita I</td>
      <td>Taladro de Mizuniya I</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Aspa ósea I</td>
      <td>Hachas óseas I</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Arco de cazador I</td>
      <td>Hachas óseas I</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Arco de Quematrice III</td>
      <td>Sílex Quematrice III</td>
      <td>1.0</td>
    </tr>
  </tbody>
</table>
</div>



```python
# Items de los pares con Jaccard = 1.0
query_jaccard_1 = """
MATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)
WHERE s.weight = 1.0 AND id(w1) < id(w2)

OPTIONAL MATCH (wc1:WeaponCraft)-[:PRODUCES]->(w1)
OPTIONAL MATCH (wc1)-[:NEEDS]->(ic1:Item)
OPTIONAL MATCH (wu1:WeaponUpgrade)-[:TRANSFORMS]->(w1)
OPTIONAL MATCH (wu1)-[:NEEDS]->(iu1:Item)

WITH w1, w2, s,
     collect(DISTINCT ic1.name) + collect(DISTINCT iu1.name) AS items_comunes

RETURN w1.name AS arma1, w2.name AS arma2, 
       s.weight AS jaccard, items_comunes
ORDER BY jaccard DESC
LIMIT 10
"""
df_top = pd.DataFrame(run_query(query_jaccard_1))
print("Top 10 pares con Jaccard = 1.0 y sus items:")
display(df_top)


```

    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 26, offset: 68} for query: '\nMATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)\nWHERE s.weight = 1.0 AND id(w1) < id(w2)\n\nOPTIONAL MATCH (wc1:WeaponCraft)-[:PRODUCES]->(w1)\nOPTIONAL MATCH (wc1)-[:NEEDS]->(ic1:Item)\nOPTIONAL MATCH (wu1:WeaponUpgrade)-[:TRANSFORMS]->(w1)\nOPTIONAL MATCH (wu1)-[:NEEDS]->(iu1:Item)\n\nWITH w1, w2, s,\n     collect(DISTINCT ic1.name) + collect(DISTINCT iu1.name) AS items_comunes\n\nRETURN w1.name AS arma1, w2.name AS arma2, \n       s.weight AS jaccard, items_comunes\nORDER BY jaccard DESC\nLIMIT 10\n'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 35, offset: 77} for query: '\nMATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)\nWHERE s.weight = 1.0 AND id(w1) < id(w2)\n\nOPTIONAL MATCH (wc1:WeaponCraft)-[:PRODUCES]->(w1)\nOPTIONAL MATCH (wc1)-[:NEEDS]->(ic1:Item)\nOPTIONAL MATCH (wu1:WeaponUpgrade)-[:TRANSFORMS]->(w1)\nOPTIONAL MATCH (wu1)-[:NEEDS]->(iu1:Item)\n\nWITH w1, w2, s,\n     collect(DISTINCT ic1.name) + collect(DISTINCT iu1.name) AS items_comunes\n\nRETURN w1.name AS arma1, w2.name AS arma2, \n       s.weight AS jaccard, items_comunes\nORDER BY jaccard DESC\nLIMIT 10\n'


    Top 10 pares con Jaccard = 1.0 y sus items:



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>arma1</th>
      <th>arma2</th>
      <th>jaccard</th>
      <th>items_comunes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Cartuchera Chata III</td>
      <td>Garrote Chata III</td>
      <td>1.0</td>
      <td>[Escama de Chatacabra+, Certificado Chatacabra S, Mandíbula de Chatacabra+, Mandíbula de Chatacabra+, Escama de Chatacabra+, Certificado Chatacabra S]</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Cartuchera Chata III</td>
      <td>Mazo Chata III</td>
      <td>1.0</td>
      <td>[Escama de Chatacabra+, Certificado Chatacabra S, Mandíbula de Chatacabra+, Mandíbula de Chatacabra+, Escama de Chatacabra+, Certificado Chatacabra S]</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Cartuchera Chata III</td>
      <td>Segur Chata III</td>
      <td>1.0</td>
      <td>[Escama de Chatacabra+, Certificado Chatacabra S, Mandíbula de Chatacabra+, Mandíbula de Chatacabra+, Escama de Chatacabra+, Certificado Chatacabra S]</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Cartuchera Chata III</td>
      <td>Glaive Chata III</td>
      <td>1.0</td>
      <td>[Escama de Chatacabra+, Certificado Chatacabra S, Mandíbula de Chatacabra+, Mandíbula de Chatacabra+, Escama de Chatacabra+, Certificado Chatacabra S]</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Punzadora potente II</td>
      <td>Golba vandálica II</td>
      <td>1.0</td>
      <td>[Garra de Congalala+, Piel de Conga+, Certificado Congalala S, Colmillo de Congalala+, Colmillo de Congalala+, Certificado Congalala S, Piel de Conga+, Garra de Congalala+]</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Punzadora potente II</td>
      <td>Maracas rosas II</td>
      <td>1.0</td>
      <td>[Garra de Congalala+, Piel de Conga+, Certificado Congalala S, Colmillo de Congalala+, Colmillo de Congalala+, Certificado Congalala S, Piel de Conga+, Garra de Congalala+]</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Punzadora potente II</td>
      <td>Bongo de guerra II</td>
      <td>1.0</td>
      <td>[Garra de Congalala+, Piel de Conga+, Certificado Congalala S, Colmillo de Congalala+, Colmillo de Congalala+, Certificado Congalala S, Piel de Conga+, Garra de Congalala+]</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Punzadora potente II</td>
      <td>Arco poder salvaje II</td>
      <td>1.0</td>
      <td>[Garra de Congalala+, Piel de Conga+, Certificado Congalala S, Colmillo de Congalala+, Colmillo de Congalala+, Certificado Congalala S, Piel de Conga+, Garra de Congalala+]</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Punzadora potente II</td>
      <td>Conga de asalto II</td>
      <td>1.0</td>
      <td>[Garra de Congalala+, Piel de Conga+, Certificado Congalala S, Colmillo de Congalala+, Colmillo de Congalala+, Certificado Congalala S, Piel de Conga+, Garra de Congalala+]</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Arco Nihil II</td>
      <td>Hacha espada Nihil II</td>
      <td>1.0</td>
      <td>[Fluido cerebral Xu Wu, Garra de Xu Wu+, Certificado Xu Wu S, Colmillo de Xu Wu+, Garra de Xu Wu+, Colmillo de Xu Wu+, Fluido cerebral Xu Wu, Certificado Xu Wu S]</td>
    </tr>
  </tbody>
</table>
</div>



```python
# Los 10 pares con Jaccard más bajo y sus items
query_jaccard_low = """
MATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)
WHERE id(w1) < id(w2)

OPTIONAL MATCH (wc1:WeaponCraft)-[:PRODUCES]->(w1)
OPTIONAL MATCH (wc1)-[:NEEDS]->(ic1:Item)

OPTIONAL MATCH (wu1:WeaponUpgrade)-[:TRANSFORMS]->(w1)
OPTIONAL MATCH (wu1)-[:NEEDS]->(iu1:Item)

OPTIONAL MATCH (wc2:WeaponCraft)-[:PRODUCES]->(w2)
OPTIONAL MATCH (wc2)-[:NEEDS]->(ic2:Item)

OPTIONAL MATCH (wu2:WeaponUpgrade)-[:TRANSFORMS]->(w2)
OPTIONAL MATCH (wu2)-[:NEEDS]->(iu2:Item)

WITH w1, w2, s,
     collect(DISTINCT ic1.name) + collect(DISTINCT iu1.name) AS raw_items_1,
     collect(DISTINCT ic2.name) + collect(DISTINCT iu2.name) AS raw_items_2

WITH w1, w2, s,
     [x IN raw_items_1 WHERE x IS NOT NULL] AS temp1,
     [x IN raw_items_2 WHERE x IS NOT NULL] AS temp2

WITH w1, w2, s,
     reduce(unique1 = [], x IN temp1 |
         CASE WHEN x IN unique1 THEN unique1 ELSE unique1 + x END
     ) AS items_arma1,
     reduce(unique2 = [], x IN temp2 |
         CASE WHEN x IN unique2 THEN unique2 ELSE unique2 + x END
     ) AS items_arma2

WITH w1, w2, s, items_arma1, items_arma2,
     [x IN items_arma1 WHERE x IN items_arma2] AS items_comunes

RETURN
    w1.name AS arma1,
    items_arma1,
    w2.name AS arma2,
    items_arma2,
    items_comunes,
    round(s.weight * 100) / 100 AS jaccard

ORDER BY jaccard ASC
LIMIT 10
"""
df_low = pd.DataFrame(run_query(query_jaccard_low))
print("\nTop 10 pares con Jaccard más bajo y sus items:")
display(df_low)
```

    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 7, offset: 49} for query: '\nMATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)\nWHERE id(w1) < id(w2)\n\nOPTIONAL MATCH (wc1:WeaponCraft)-[:PRODUCES]->(w1)\nOPTIONAL MATCH (wc1)-[:NEEDS]->(ic1:Item)\n\nOPTIONAL MATCH (wu1:WeaponUpgrade)-[:TRANSFORMS]->(w1)\nOPTIONAL MATCH (wu1)-[:NEEDS]->(iu1:Item)\n\nOPTIONAL MATCH (wc2:WeaponCraft)-[:PRODUCES]->(w2)\nOPTIONAL MATCH (wc2)-[:NEEDS]->(ic2:Item)\n\nOPTIONAL MATCH (wu2:WeaponUpgrade)-[:TRANSFORMS]->(w2)\nOPTIONAL MATCH (wu2)-[:NEEDS]->(iu2:Item)\n\nWITH w1, w2, s,\n     collect(DISTINCT ic1.name) + collect(DISTINCT iu1.name) AS raw_items_1,\n     collect(DISTINCT ic2.name) + collect(DISTINCT iu2.name) AS raw_items_2\n\nWITH w1, w2, s,\n     [x IN raw_items_1 WHERE x IS NOT NULL] AS temp1,\n     [x IN raw_items_2 WHERE x IS NOT NULL] AS temp2\n\nWITH w1, w2, s,\n     reduce(unique1 = [], x IN temp1 |\n         CASE WHEN x IN unique1 THEN unique1 ELSE unique1 + x END\n     ) AS items_arma1,\n     reduce(unique2 = [], x IN temp2 |\n         CASE WHEN x IN unique2 THEN unique2 ELSE unique2 + x END\n     ) AS items_arma2\n\nWITH w1, w2, s, items_arma1, items_arma2,\n     [x IN items_arma1 WHERE x IN items_arma2] AS items_comunes\n\nRETURN\n    w1.name AS arma1,\n    items_arma1,\n    w2.name AS arma2,\n    items_arma2,\n    items_comunes,\n    round(s.weight * 100) / 100 AS jaccard\n\nORDER BY jaccard ASC\nLIMIT 10\n'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 16, offset: 58} for query: '\nMATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)\nWHERE id(w1) < id(w2)\n\nOPTIONAL MATCH (wc1:WeaponCraft)-[:PRODUCES]->(w1)\nOPTIONAL MATCH (wc1)-[:NEEDS]->(ic1:Item)\n\nOPTIONAL MATCH (wu1:WeaponUpgrade)-[:TRANSFORMS]->(w1)\nOPTIONAL MATCH (wu1)-[:NEEDS]->(iu1:Item)\n\nOPTIONAL MATCH (wc2:WeaponCraft)-[:PRODUCES]->(w2)\nOPTIONAL MATCH (wc2)-[:NEEDS]->(ic2:Item)\n\nOPTIONAL MATCH (wu2:WeaponUpgrade)-[:TRANSFORMS]->(w2)\nOPTIONAL MATCH (wu2)-[:NEEDS]->(iu2:Item)\n\nWITH w1, w2, s,\n     collect(DISTINCT ic1.name) + collect(DISTINCT iu1.name) AS raw_items_1,\n     collect(DISTINCT ic2.name) + collect(DISTINCT iu2.name) AS raw_items_2\n\nWITH w1, w2, s,\n     [x IN raw_items_1 WHERE x IS NOT NULL] AS temp1,\n     [x IN raw_items_2 WHERE x IS NOT NULL] AS temp2\n\nWITH w1, w2, s,\n     reduce(unique1 = [], x IN temp1 |\n         CASE WHEN x IN unique1 THEN unique1 ELSE unique1 + x END\n     ) AS items_arma1,\n     reduce(unique2 = [], x IN temp2 |\n         CASE WHEN x IN unique2 THEN unique2 ELSE unique2 + x END\n     ) AS items_arma2\n\nWITH w1, w2, s, items_arma1, items_arma2,\n     [x IN items_arma1 WHERE x IN items_arma2] AS items_comunes\n\nRETURN\n    w1.name AS arma1,\n    items_arma1,\n    w2.name AS arma2,\n    items_arma2,\n    items_comunes,\n    round(s.weight * 100) / 100 AS jaccard\n\nORDER BY jaccard ASC\nLIMIT 10\n'


    
    Top 10 pares con Jaccard más bajo y sus items:



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>arma1</th>
      <th>items_arma1</th>
      <th>arma2</th>
      <th>items_arma2</th>
      <th>items_comunes</th>
      <th>jaccard</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Decapitagallinas I</td>
      <td>[Carbalita, Escama de Kut-Ku+, Oreja de Kut-Ku, Certificado Yian Kut-Ku S, Ala Kut-Ku]</td>
      <td>Lanza de paladín I</td>
      <td>[Vale de la comisión, Piedra de lava, Gracio, Carbalita]</td>
      <td>[Carbalita]</td>
      <td>0.13</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Llama chthonia II</td>
      <td>[Certificado Ébano G. S, Sangre de Guardián+, Placa de Ébano G., Garra de Ébano G.+]</td>
      <td>Agraviosonora Dosha II</td>
      <td>[Certificado Doshaguma G. S, Sangre de Guardián+, Colmillo Doshaguma G.+, Garra Doshaguma G.+]</td>
      <td>[Sangre de Guardián+]</td>
      <td>0.14</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Desolador Dosha II</td>
      <td>[Garra Doshaguma G.+, Sangre de Guardián+, Colmillo Doshaguma G.+, Certificado Doshaguma G. S]</td>
      <td>Vajra chthonia II</td>
      <td>[Sangre de Guardián+, Placa de Ébano G., Certificado Ébano G. S, Garra de Ébano G.+]</td>
      <td>[Sangre de Guardián+]</td>
      <td>0.14</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Llama chthonia II</td>
      <td>[Certificado Ébano G. S, Sangre de Guardián+, Placa de Ébano G., Garra de Ébano G.+]</td>
      <td>Azotegrima Dosha II</td>
      <td>[Garra Doshaguma G.+, Sangre de Guardián+, Colmillo Doshaguma G.+, Certificado Doshaguma G. S]</td>
      <td>[Sangre de Guardián+]</td>
      <td>0.14</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Llama chthonia II</td>
      <td>[Certificado Ébano G. S, Sangre de Guardián+, Placa de Ébano G., Garra de Ébano G.+]</td>
      <td>Rompegigas Dosha II</td>
      <td>[Colmillo Doshaguma G.+, Sangre de Guardián+, Garra Doshaguma G.+, Certificado Doshaguma G. S]</td>
      <td>[Sangre de Guardián+]</td>
      <td>0.14</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Llama chthonia II</td>
      <td>[Certificado Ébano G. S, Sangre de Guardián+, Placa de Ébano G., Garra de Ébano G.+]</td>
      <td>Azotaguardia Dosha II</td>
      <td>[Colmillo Doshaguma G.+, Garra Doshaguma G.+, Sangre de Guardián+, Certificado Doshaguma G. S]</td>
      <td>[Sangre de Guardián+]</td>
      <td>0.14</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Llama chthonia II</td>
      <td>[Certificado Ébano G. S, Sangre de Guardián+, Placa de Ébano G., Garra de Ébano G.+]</td>
      <td>Horadatiniebla Dosha II</td>
      <td>[Sangre de Guardián+, Colmillo Doshaguma G.+, Certificado Doshaguma G. S, Garra Doshaguma G.+]</td>
      <td>[Sangre de Guardián+]</td>
      <td>0.14</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Desolador Dosha II</td>
      <td>[Garra Doshaguma G.+, Sangre de Guardián+, Colmillo Doshaguma G.+, Certificado Doshaguma G. S]</td>
      <td>Llama chthonia II</td>
      <td>[Certificado Ébano G. S, Sangre de Guardián+, Placa de Ébano G., Garra de Ébano G.+]</td>
      <td>[Sangre de Guardián+]</td>
      <td>0.14</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Llama chthonia II</td>
      <td>[Certificado Ébano G. S, Sangre de Guardián+, Placa de Ébano G., Garra de Ébano G.+]</td>
      <td>Rompetumbas Dosha II</td>
      <td>[Garra Doshaguma G.+, Colmillo Doshaguma G.+, Sangre de Guardián+, Certificado Doshaguma G. S]</td>
      <td>[Sangre de Guardián+]</td>
      <td>0.14</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Llama chthonia II</td>
      <td>[Certificado Ébano G. S, Sangre de Guardián+, Placa de Ébano G., Garra de Ébano G.+]</td>
      <td>Cortasangre Dosha II</td>
      <td>[Certificado Doshaguma G. S, Garra Doshaguma G.+, Colmillo Doshaguma G.+, Sangre de Guardián+]</td>
      <td>[Sangre de Guardián+]</td>
      <td>0.14</td>
    </tr>
  </tbody>
</table>
</div>


---
## Paso 2: Detección de comunidades con el algoritmo de Leiden
n?
Leiden es un algoritmo de detección de comunidades que mejora a Louvain. Garantiza que las comunidades estén bien conectadas internamente y tiene mejor calidad de partición (modularity). Es ideal para nuestro caso porque:
- Las armas forman grupos naturales por tipo/serie
- El peso Jaccard nos da una medida de similaridad continua
- Leiden respeta mejor los pesos de las aristas que Louvain

### Proceso GDS
1. **Proyectar** el grafo en memoria (solo nodos Weapon y relaciones SIMILAR)
2. **Ejecutar** Leiden en modo `write` para que escriba el resultado en cada nodo
3. **Verificar** la distribución de comunidades


```python
#Eliminar proyección previa si existe
try:
    gds.graph.drop(gds.graph.get("weapons_graph"))
    print("Proyección previa eliminada.")
except Exception:
    print("No había proyección previa.")
```

    No había proyección previa.



```python
#Proyectar el subgrafo de armas en memoria GDS

# Proyectamos:
#   - Nodos: Weapon
#   - Relaciones: SIMILAR (con su propiedad weight)
# Usamos UNDIRECTED porque la similaridad es simétrica.

G, projection_result = gds.graph.project(
    "weapons_graph",        # nombre de la proyección
    "Weapon",              # etiqueta de nodos
    {
        "SIMILAR": {
            "orientation": "UNDIRECTED",
            "properties": ["weight"]
        }
    }
)

print(f"Grafo proyectado:")
print(f"  Nodos:     {projection_result['nodeCount']}")
print(f"  Relaciones: {projection_result['relationshipCount']}")
```

    Grafo proyectado:
      Nodos:     1024
      Relaciones: 1150



```python
leiden_result = gds.leiden.write(
    G,
    writeProperty="communityId",
    relationshipWeightProperty="weight",
    maxLevels=10,
    gamma=1.0,
    theta=0.01
)

print(f"Leiden ejecutado:")
print(f"  Comunidades encontradas:{leiden_result['communityCount']}")
print(f"  Modularity:{leiden_result['modularity']:.4f}")
print(f"  Niveles ejecutados:{leiden_result['ranLevels']}")
```

    Leiden ejecutado:
      Comunidades encontradas:903
      Modularity:0.8940
      Niveles ejecutados:2



```python
# Verificar distribución de comunidades
dist_query = """
MATCH (w:Weapon)
WHERE w.communityId IS NOT NULL
RETURN w.communityId AS comunidad,
       count(*) AS num_armas
ORDER BY num_armas DESC
LIMIT 15
"""
df_dist = pd.DataFrame(run_query(dist_query))
print("Distribución de armas por comunidad, lo top 15:")
display(df_dist)
```

    Distribución de armas por comunidad, lo top 15:



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>comunidad</th>
      <th>num_armas</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>579</td>
      <td>14</td>
    </tr>
    <tr>
      <th>1</th>
      <td>572</td>
      <td>14</td>
    </tr>
    <tr>
      <th>2</th>
      <td>570</td>
      <td>14</td>
    </tr>
    <tr>
      <th>3</th>
      <td>651</td>
      <td>11</td>
    </tr>
    <tr>
      <th>4</th>
      <td>734</td>
      <td>11</td>
    </tr>
    <tr>
      <th>5</th>
      <td>622</td>
      <td>10</td>
    </tr>
    <tr>
      <th>6</th>
      <td>779</td>
      <td>8</td>
    </tr>
    <tr>
      <th>7</th>
      <td>558</td>
      <td>7</td>
    </tr>
    <tr>
      <th>8</th>
      <td>590</td>
      <td>7</td>
    </tr>
    <tr>
      <th>9</th>
      <td>560</td>
      <td>7</td>
    </tr>
    <tr>
      <th>10</th>
      <td>676</td>
      <td>6</td>
    </tr>
    <tr>
      <th>11</th>
      <td>600</td>
      <td>6</td>
    </tr>
    <tr>
      <th>12</th>
      <td>607</td>
      <td>5</td>
    </tr>
    <tr>
      <th>13</th>
      <td>615</td>
      <td>5</td>
    </tr>
    <tr>
      <th>14</th>
      <td>699</td>
      <td>5</td>
    </tr>
  </tbody>
</table>
</div>


---
## Paso 3: PageRank por comunidad — encontrar el arma representante
k?
PageRank mide la **centralidad** de un nodo en el grafo. Un arma con PageRank alto dentro de su comunidad es aquella que:
- Está conectada a muchas otras armas similares
- Sus vecinos también están bien conectados

Esto la hace candidata natural como **representante** de la comunidad, ya que es el arma que más materiales comparte con el resto de armas de su strategia
GDS no permite hacer PageRank por subgrafo de comunidad directamente con filtros por propiedad en la proyección de forma trivial. La estrategia más limpia es:
1. Para cada comunidad, filtrar los nodos con esa `communityId`
2. Crear una proyección filtrada usando `gds.graph.filter` o hacer el cálculo vía Cypher
3. Aplicar PageRank sobre esa proyección

Usaremos `gds.graph.filter` para extraer subgrafos por comunidad y aplicar PageRank a cada uno.


```python
# Obtener la lista de comunidades
communities_query = """
MATCH (w:Weapon)
WHERE w.communityId IS NOT NULL
RETURN DISTINCT w.communityId AS comunidad, count(*) AS num_armas
ORDER BY num_armas DESC
"""
communities = run_query(communities_query)
print(f"Total de comunidades: {len(communities)}")
community_ids = [c['comunidad'] for c in communities]
```

    Total de comunidades: 903



```python
# Eliminar proyección anterior
try:
    gds.graph.drop(G)
    print("Proyección anterior eliminada")
except:
    pass

# Re-proyectar incluyendo communityId como propiedad del nodo
G, stats = gds.graph.project(
    "weapons_graph",
    {"Weapon": {"properties": ["communityId"]}},
    {"SIMILAR": {"orientation": "UNDIRECTED", "properties": ["weight"]}}
)
print(f"Nodos: {stats['nodeCount']}, Relaciones: {stats['relationshipCount']}")
print("Proyección actualizada con communityId")
```

    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'


    Proyección anterior eliminada
    Nodos: 1024, Relaciones: 1150
    Proyección actualizada con communityId



```python
# Para comunidades con 1 solo nodo, ese nodo es trivialmente el representante
# Para comunidades con 2+ nodos, aplicamos PageRank

representatives = [] #{comunidad, arma_representante, pagerank}

for community_data in communities:
    community_id = community_data['comunidad']
    num_armas = community_data['num_armas']
    
    if num_armas == 1:
        # Comunidad de un solo nodo: es el representante por defecto
        single_query = """
        MATCH (w:Weapon {communityId: $cid})
        RETURN w.name AS nombre
        """
        result = run_query(single_query, {"cid": community_id})
        representatives.append({
            'comunidad': community_id,
            'representante': result[0]['nombre'],
            'pagerank': 1.0,
            'num_armas': num_armas
        })
        continue
    
    # Crear subgrafo filtrado para esta comunidad
    subgraph_name = f"community_{community_id}"
    
    try:
        # Filtrar el grafo principal por communityId
        H, _ = gds.graph.filter(
            subgraph_name,
            G,
            f"n.communityId = {community_id}",
            "*"  # todas las relaciones entre nodos filtrados
        )
        
        # Ejecutar PageRank en el subgrafo
        # dampingFactor=0.85 es el valor estándar
        # maxIterations=20 es suficiente para grafos pequeños
        pr_result = gds.pageRank.stream(
            H,
            maxIterations=20,
            dampingFactor=0.85,
            relationshipWeightProperty="weight"
        )
        
        # Obtener el nodo con mayor PageRank
        top_node = pr_result.sort_values("score", ascending=False).iloc[0]
        
        # Obtener el nombre del arma (nodeId es el id interno de GDS)
        name_query = """
        MATCH (w:Weapon)
        WHERE id(w) = $node_id
        RETURN w.name AS nombre
        """
        name_result = run_query(name_query, {"node_id": int(top_node['nodeId'])})
        
        if name_result:
            representatives.append({
                'comunidad': community_id,
                'representante': name_result[0]['nombre'],
                'pagerank': round(float(top_node['score']), 4),
                'num_armas': num_armas
            })
        
        # Limpiar subgrafo
        gds.graph.drop(H)
        
    except Exception as e:
        print(f"Error en comunidad {community_id}: {e}")
        # Intentamos limpiar igualmente
        try:
            gds.graph.drop(gds.graph.get(subgraph_name))
        except:
            pass

print(f"\nRepresentantes calculados para {len(representatives)} comunidades.")
```

    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 3, column: 15, offset: 40} for query: '\n        MATCH (w:Weapon)\n        WHERE id(w) = $node_id\n        RETURN w.name AS nombre\n        '
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The procedure has a deprecated field. ('schema' returned by 'gds.graph.drop' is deprecated.)} {position: line: 1, column: 1, offset: 0} for query: 'CALL gds.graph.drop($graph_name, $fail_if_missing, $db_name)'


    
    Representantes calculados para 903 comunidades.



```python
df_reps = pd.DataFrame(representatives)
df_reps_multi = df_reps[df_reps['num_armas'] > 1].sort_values('num_armas', ascending=False)


print("ARMAS REPRESENTANTES POR COMUNIDAD (comunidades con >1 arma)")
for _, row in df_reps_multi.iterrows():
    print(f"  Comunidad {row['comunidad']:4d} | {row['num_armas']:3d} armas | "
          f"PageRank: {row['pagerank']:.4f} | Representante: {row['representante']}")

print(f"\nTotal comunidades con >1 arma: {len(df_reps_multi)}")
print(f"Total comunidades singleton:   {len(df_reps[df_reps['num_armas'] == 1])}")
```

    ARMAS REPRESENTANTES POR COMUNIDAD (comunidades con >1 arma)
      Comunidad  579 |  14 armas | PageRank: 0.9612 | Representante: Arco de cazador I
      Comunidad  570 |  14 armas | PageRank: 0.9612 | Representante: Zoh Samira I
      Comunidad  572 |  14 armas | PageRank: 0.9612 | Representante: Lluvia bendita I
      Comunidad  734 |  11 armas | PageRank: 0.9612 | Representante: Ira Reina I
      Comunidad  651 |  11 armas | PageRank: 1.0874 | Representante: Decapitagallinas I
      Comunidad  622 |  10 armas | PageRank: 1.0805 | Representante: Desolador Dosha II
      Comunidad  779 |   8 armas | PageRank: 0.9612 | Representante: Filo venenoso I
      Comunidad  560 |   7 armas | PageRank: 0.9612 | Representante: Belicistas Gravios I
      Comunidad  558 |   7 armas | PageRank: 0.9612 | Representante: Blan Sharanga I
      Comunidad  590 |   7 armas | PageRank: 0.9612 | Representante: Arco de Quematrice III
      Comunidad  676 |   6 armas | PageRank: 0.9612 | Representante: Punzadora potente II
      Comunidad  600 |   6 armas | PageRank: 0.9612 | Representante: Balaexplosión III
      Comunidad  699 |   5 armas | PageRank: 0.9612 | Representante: Cartuchera Chata III
      Comunidad  615 |   5 armas | PageRank: 0.9612 | Representante: Arco huracán II
      Comunidad  607 |   5 armas | PageRank: 0.9612 | Representante: Romposcudo II
      Comunidad  703 |   5 armas | PageRank: 0.9612 | Representante: Falces Barina III
      Comunidad  465 |   3 armas | PageRank: 0.9612 | Representante: Arco Nihil II
    
    Total comunidades con >1 arma: 17
    Total comunidades singleton:   886



```python
from yfiles_jupyter_graphs import GraphWidget

# Consultamos algunos nodos y relaciones para visualizar
# (limitamos a una comunidad grande para que sea legible)
query_graph = """
MATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)
WHERE w1.communityId = w2.communityId
WITH w1.communityId AS cid, count(*) AS tam
ORDER BY tam DESC
LIMIT 1

MATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)
WHERE w1.communityId = cid AND w2.communityId = cid
AND id(w1) < id(w2)
RETURN w1.name AS arma1, w1.communityId AS com1,
       w2.name AS arma2, w2.communityId AS com2,
       s.weight AS peso
LIMIT 50
"""

rows = run_query(query_graph)

# Construimos nodos y aristas para yfiles
nodes = {}
edges = []

for row in rows:
    if row['arma1'] not in nodes:
        nodes[row['arma1']] = {"id": row['arma1'], "properties": {"name": row['arma1'], "community": row['com1']}}
    if row['arma2'] not in nodes:
        nodes[row['arma2']] = {"id": row['arma2'], "properties": {"name": row['arma2'], "community": row['com2']}}
    edges.append({
        "start": row['arma1'],
        "end": row['arma2'],
        "properties": {"weight": round(row['peso'], 2)}
    })

w = GraphWidget()
w.nodes = list(nodes.values())
w.edges = edges
display(w)
```

    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 10, column: 5, offset: 250} for query: '\nMATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)\nWHERE w1.communityId = w2.communityId\nWITH w1.communityId AS cid, count(*) AS tam\nORDER BY tam DESC\nLIMIT 1\n\nMATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)\nWHERE w1.communityId = cid AND w2.communityId = cid\nAND id(w1) < id(w2)\nRETURN w1.name AS arma1, w1.communityId AS com1,\n       w2.name AS arma2, w2.communityId AS com2,\n       s.weight AS peso\nLIMIT 50\n'
    Received notification from DBMS server: {severity: WARNING} {code: Neo.ClientNotification.Statement.FeatureDeprecationWarning} {category: DEPRECATION} {title: This feature is deprecated and will be removed in future versions.} {description: The query used a deprecated function. ('id' has been replaced by 'elementId or consider using an application-generated id')} {position: line: 10, column: 14, offset: 259} for query: '\nMATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)\nWHERE w1.communityId = w2.communityId\nWITH w1.communityId AS cid, count(*) AS tam\nORDER BY tam DESC\nLIMIT 1\n\nMATCH (w1:Weapon)-[s:SIMILAR]-(w2:Weapon)\nWHERE w1.communityId = cid AND w2.communityId = cid\nAND id(w1) < id(w2)\nRETURN w1.name AS arma1, w1.communityId AS com1,\n       w2.name AS arma2, w2.communityId AS com2,\n       s.weight AS peso\nLIMIT 50\n'



    GraphWidget(layout=Layout(height='640px', width='100%'))



```python

```
