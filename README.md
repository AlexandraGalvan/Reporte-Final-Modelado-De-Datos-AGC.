El siguiente repositorio es el reporte final de la asignatura de Modelado de Datos de la Maestría en Cómputo Aplicado de la Universidad de Guadalajara elaborado por **Alexandra Galván Chávez.**

# Recomendador de productos en tiempo real con Neo4j y Java.
## 🏪 Contexto general.
Una tienda en línea que ofrece miles de productos diversos tales como ropa, gadgets y accesorios recibe un buen nivel de tráfico. Sin embargo, sus métricas de conversión son bajas: muchos usuarios navegan por la página, consultan algunos productos y finalmente se retiran sin realizar compras. El equipo de análisis de datos identificó que la sección de recomendaciones de productos presenta artículos genéricos, sin ningún tipo de personalización para los usuarios.

## 🧐 El problema.
El sistema de recomendación está basado en consultas SQL tradicionales sobre una base relacional (MySQL). El modelo analiza datos individuales (producto-categoría-precio) sin contexto ni información entre los usuarios y los patrones de navegación. Las consultas SQL con múltiples joins se vuelven lentas a medida que crece la base de datos (>10M filas) y no hay forma fácil de representar relaciones complejas como *usuarios que compraron A y vieron B compraron C.* Los síntomas observados son que los tiempos de respuesta de recomendación son de alrededor de 1.2 segundos por consulta, el CTR (Click Through Rate) de recomendaciones es de apenas 1.5% y el AOV (Valor Promedio de Compra) es de $580.00 MXN.

## 🚀 Solución propuesta.
Desarrollar e implementar un motor de recomendación basado en grafos utilizando Neo4j, capaz de modelar de manera integral las relaciones entre usuarios, productos, categorías y eventos de interacción, como compras, vistas y *likes*. Este enfoque permitirá identificar patrones de comportamiento en tiempo real, proporcionando recomendaciones personalizadas y contextuales que se adapten a los intereses y necesidades de cada usuario, mejorando así la experiencia de navegación y las métricas de conversión de la tienda en línea.

## ⚙️ Stack tecnológico.
- **Lenguaje:** Java 21.
- **Framework:** Spring Boot.
- **BD:** Neo4j (Community/Enterprise).
- **Driver:** Spring Data Neo4j + Neo4j Java Driver.
- **Caché:** Redis (via Lettuce) para respuestas frecuentes.
- **Ingesta:** Apache Kafka (para flujos de eventos de compra/navegación).
- **Build:** Maven o Gradle.

## 🧠 Modelado del grafo.
Los nodos principales son los siguientes:
- *(:User {id, nombre, edad, ciudad})*
- *(:Product {id, nombre, categoría, precio})*
- *(:Category {id, nombre})*
- *(:Event {tipo, timestamp})*

Las relaciones son las siguientes:
- *(User)-[:VIEWED]->(Product)*
- *(User)-[:BOUGHT]->(Product)*
- *(Product)-[:BELONGS_TO]->(Category)*
- *(Product)-[:SIMILAR_TO]->(Product)* (Calculada según características).

# 🧩 Implementación técnica.
- **Ingesta de eventos:** Un microservicio Java (Spring Boot) consume mensajes de Kafka (view_event, purchase_event), los transforma y los inserta en Neo4j.
```java
@Service
public class EventIngestService {
    @Autowired private Neo4jClient neo4jClient;
    public void registerPurchase(String userId, String productId) {
        String query = "MERGE (u:User {id:$u}) MERGE (p:Product {id:$p}) " +
                       "MERGE (u)-[:BOUGHT {timestamp:datetime()}]->(p)";
        neo4jClient.query(query).bindAll(Map.of("u", userId, "p", productId)).run();
    }
}
```

- **Recomendador para la consulta personalizada:** Usando Cypher.
```cypher
MATCH (u:User {id:$userId})-[:BOUGHT]->(p1)<-[:BOUGHT]-(other:User)-[:BOUGHT]->(rec:Product)
WHERE NOT (u)-[:BOUGHT]->(rec)
RETURN rec, COUNT(*) AS score
ORDER BY score DESC LIMIT 5;
```

- **Caché en Redis:** Los resultados se almacenan con TTL de diez minutos.
```java
redisTemplate.opsForValue().set("recs:"+userId, recommendations, 10, TimeUnit.MINUTES);
```

- **API REST:** Endpoint /api/recommendations/{userId} entrega productos recomendados en JSON.
- **Interfaz demo:** Frontend simple (React o Thymeleaf) que muestra los productos recomendados y su tiempo de respuesta.

# 📈 Impacto y métricas después de la solución.
Tras un mes de implementación del piloto en una categoría específica de productos, se observaron mejoras significativas en los principales indicadores de rendimiento.
- **CTR en recomendaciones:** pasó de 1.5% a 6.8%, evidenciando un aumento considerable en la interacción de los usuarios con los productos sugeridos.
- **Valor promedio de compra (AOV):** incrementó de $580 a $715 MXN, lo que representa un crecimiento del 23%.
- **Tiempo promedio de respuesta del sistema:** se redujo de 1.2 segundos a 80 ms gracias a la implementación de caché en Redis, optimizando la experiencia del usuario.
- **Engagement:** las sesiones con al menos un clic en recomendaciones aumentaron en un 35%, reflejando una mayor participación y relevancia del sistema de recomendación.
