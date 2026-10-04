# Post-contenido Unidad 7: Patrones Arquitectónicos I

## Descripción
Repositorio del post-contenido de la Unidad 7 de Patrones de Diseño de Software. Un único proyecto Spring Boot (`multas-biblioteca-api`) para la gestión de multas de biblioteca universitaria, estructurado en dos partes: una API REST en capas sobre H2 y una extensión con arquitectura hexagonal para el pago en línea de multas mediante pasarelas intercambiables.

## Parte 1: Arquitectura en Capas
Se implementó una API REST aplicando arquitectura en capas:
- **Capa Presentación (`controller/`)**: `MultaController` expone los endpoints REST bajo `/api/multas` y maneja errores centralizados con `GlobalExceptionHandler`.
- **Capa Aplicación (`service/`)**: `MultaService` concentra la orquestación y lógica de negocio (límite de multas pendientes).
- **Capa Dominio (`model/`)**: La entidad `Multa` encapsula el cálculo del monto (`Multa.calcularMonto`) según días de atraso y el cambio de estado.
- **Capa Infraestructura (`repository/`)**: `MultaRepository` extiende `JpaRepository` e incluye la consulta agregada `countByEstudianteIdAndEstado`.

## Parte 2: Pago en Línea con Dos Pasarelas
Se eligió la **Opción C (Puerto de dominio con adaptadores - Arquitectura Hexagonal en la porción de pago)**.
- **`domain/port/PasarelaPagoPort`** y **`domain/ResultadoPago`**: Definidos en Java puro, sin anotaciones ni dependencias de Spring Boot ni HTTP.
- **`infrastructure/pago/PagosUdesAdapter`** e **`infrastructure/pago/WompiAdapter`**: Adaptadores que traducen las respuestas HTTP específicas (PagosUDES con `idTransaccion/estadoTransaccion` y Wompi con centavos y `reference/status`) al modelo agnóstico `ResultadoPago`.
- **Selección por Configuración**: Con `@ConditionalOnProperty(prefix = "app.pagos", name = "proveedor")` se activa el adaptador correspondiente en tiempo de arranque mediante `application.properties` sin modificar `MultaService` ni `MultaController`.

## Cómo Ejecutar
```bash
cd multas-biblioteca-api
mvn spring-boot:run

Herramientas Utilizadas
Java 17

Spring Boot 3.x (Spring Web, Spring Data JPA, Validation)

Base de Datos H2

RestTemplate

Apache Maven

Git / GitHub

Decisiones de Diseño
Punto de Decisión 1: Cálculo del monto: ¿Entidad o Service?
Decisión: Ubicar Multa.calcularMonto como un método de dominio estático en la entidad Multa.
Justificación: El cálculo del monto depende únicamente de un dato propio (diasAtraso) y de constantes de negocio (VALOR_POR_DIA y TOPE_MAXIMO). No requiere ninguna consulta externa ni bean de Spring. Colocarlo en MultaService habría generado un modelo de dominio anémico (entidades tratadas como meros contenedores de datos sin comportamiento) y habría forzado a cualquier cliente a depender del servicio para una regla pura del dominio.

Punto de Decisión 2: Conteo de multas pendientes: ¿Consulta o filtrado en memoria?
Decisión: Implementar countByEstudianteIdAndEstado directamente en MultaRepository.
Justificación: Delegar el conteo al motor de base de datos genera una consulta COUNT(*) eficiente. Si la regla se hubiera resuelto trayendo todas las multas del estudiante a memoria con findByEstudianteId para filtrarlas con Java Streams, el rendimiento degradaría drásticamente a medida que el historial de multas del estudiante crezca con el tiempo.

Punto de Decisión 3: Selección del adaptador activo
Decisión: Utilizar @ConditionalOnProperty para que Spring registre un único bean de PasarelaPagoPort en el contexto.
Justificación: Como el requisito de negocio establece que cada sede utiliza una pasarela fija durante la fase piloto, resolver la activación en tiempo de arranque evita condicionales o mapas en MultaService. Inyectar un Map<String, PasarelaPagoPort> habría brindado dinamismo en tiempo de ejecución, pero habría acoplado el servicio a las claves de configuración de los proveedores externos.

Punto de Decisión 4: Diseño del puerto y el tipo de resultado
Decisión: Diseñar ResultadoPago como un record agnóstico con campos universales (proveedor, exitoso, referenciaExterna, mensaje).
Justificación: Si el puerto devolviera estructuras específicas como idTransaccion de PagosUDES, el adaptador de Wompi se vería forzado a adaptar sus datos a terminología ajena o MultaService tendría que conocer los DTOs de cada proveedor. Mantener el contrato agnóstico garantiza que agregar una tercera pasarela no altere la firma del puerto ni la capa de aplicación.

Trade-off Considerado (Parte 2)
Se evaluó extender la arquitectura en capas simple mediante el patrón Strategy en la capa de servicio (Opción B) frente a introducir un puerto de dominio con adaptadores (Opción C).

Alternativa descartada: Opción B (Interfaz Strategy en service/).

Ganancia de la Opción C: Se logró un desacoplamiento absoluto de los detalles HTTP y estructuras externas respecto del dominio central. Ninguna tecnología HTTP o formato externo se filtró a MultaService ni a la lógica del negocio.

Costo adicional: Incurrió en la creación de paquetes adicionales (domain/port e infrastructure/pago), clases DTO intermedias de traducción y mayor boilerplate en comparación con mantener las interfaces dentro de la capa de servicio.

Veredicto: La inversión en la Opción C se justifica debido a que cada pasarela maneja contratos y tipos de datos incompatibles (centavos frente a montos estándar). Si el piloto terminara y la universidad estandarizara una única pasarela de pago definitiva, el equipo consideraría simplificar la estructura reduciendo la capa de adaptadores sin afectar la entidad ni la persistencia básica.

Conclusiones
El desarrollo de esta actividad permitió evidenciar cómo la arquitectura en capas resulta ideal para CRUDs y reglas de negocio centradas en la persistencia interna, manteniendo una estructura limpia y mantenible. Por otro lado, la incorporación de un puerto de dominio en la Parte 2 demostró el valor de la Arquitectura Hexagonal cuando el sistema debe interactuar con proveedores externos con contratos heterogéneos. La principal lección aprendida es que los patrones arquitectónicos no deben aplicarse por dogma, sino evaluando conscientemente los trade-offs entre complejidad y flexibilidad.

