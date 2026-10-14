# TecniHogar
## Análisis

### 1. Requisitos funcionales

#### Información a conservar
- Órdenes de servicio: código único, cliente, dirección, horas estimadas (decimal positivo), zona (Metropolitana o Periférica), estado actual, cantidad de visitas realizadas, costo cobrado y datos según categoría:
  - Diagnóstico: horas estimadas (máximo 1.5 h).
  - Instalación: cantidad de puntos (entero positivo).
  - Instalación eléctrica certificada: cantidad de puntos y valor declarado de materiales.
  - Mantenimiento aire acondicionado: capacidad en BTU (entre 9,000 y 60,000) y zona fija en Metropolitana.
- Cuadrillas: código único, capacidad máxima de horas por jornada, si está certificada en refrigeración y si tiene una jornada abierta.
- Jornadas: cuadrilla asignada, órdenes asignadas, horas estimadas acumuladas, estado y resultados de visitas.
- Ingresos: acumuladores separados para ingresos por registro, ingresos por visitas adicionales (Q40.00 por visita tras la primera) e ingresos por categoría. Ni cotizaciones ni operaciones fallidas alteran los ingresos.

#### Operaciones permitidas
1. Cotizar servicio: muestra costo y tiempo estimado sin registrar nada ni alterar ingresos.
2. Registrar orden: cobra el servicio, crea la orden en estado Pendiente con 0 visitas.
3. Registrar cuadrilla: guarda código, capacidad de horas y certificación en refrigeración.
4. Crear jornada: asocia una cuadrilla libre e inicia en estado En Preparación.
5. Asignar orden a jornada: vincula una orden pendiente si hay capacidad y compatibilidad.
6. Retirar orden de jornada: desvincula la orden y la regresa a Pendiente.
7. Iniciar jornada: pasa jornada y órdenes a En Atención. Si una orden ya tuvo visitas previas, cobra recargo de Q40.00 en ingresos.
8. Registrar resultados y cerrar jornada: califica cada orden como Completada o Fallida. Al completar todas, la jornada pasa a Cerrada y libera a la cuadrilla.
9. Consultar órdenes y cuadrillas: lista sus datos y estados actuales.
10. Generar reportes: conteo de órdenes por categoría y estado; ingresos desglosados (registro, adicionales y categoría); jornadas no cerradas con horas y capacidad disponible.

#### Condiciones de validez
- Códigos únicos y no vacíos para órdenes y cuadrillas.
- Números positivos: horas estimadas > 0, puntos > 0, capacidad cuadrilla > 0, materiales >= 0.
- Diagnóstico: horas estimadas <= 1.5.
- Aire acondicionado: solo zona Metropolitana; capacidad entre 9,000 y 60,000 BTU.
- Asignación a jornada: horas estimadas acumuladas + horas de la orden <= capacidad de cuadrilla (usando horas estimadas, no facturables); aire acondicionado requiere cuadrilla con refrigeración; orden debe estar Pendiente y no asignada a otra jornada.
- Inicio de jornada: debe tener al menos una orden y la cuadrilla no puede tener otra jornada sin cerrar.
- Cierre de jornada: todas las órdenes deben tener resultado registrado.

---

### 2. Decisiones que dependen de la categoría del servicio

1. Cálculo del costo:
- Diagnóstico (Clase Diagnostico): Q150.00 en Metropolitana y Q225.00 en Periférica, sin importar las horas.
- Instalación (Clase Instalacion): base (Q100.00 metro / Q175.00 peri) + Q90.00 por hora facturable. Las horas facturables son el mayor entre horas estimadas y puntos x 0.75, redondeado al entero superior.
- Instalación eléctrica certificada (Clase InstalacionElectricaCertificada): costo de instalación + Q120.00 de certificado + 5% del valor de materiales.
- Mantenimiento aire acondicionado (Clase MantenimientoAireAcondicionado): base Q100.00 + (horas estimadas x Q110.00) + Q80.00 de gas refrigerante.
- Clase y razón: cada subclase conoce su fórmula y tarifas específicas.

2. Días hábiles estimados de atención:
- Diagnóstico: 1 día en Metropolitana, 2 en Periférica.
- Instalación: 3 días en Metropolitana, 5 en Periférica.
- Instalación eléctrica certificada: 2 días más que la instalación normal (5 días metro, 7 peri).
- Mantenimiento aire acondicionado: siempre 2 días.
- Clase y razón: cada subclase define su promesa según complejidad técnica y logística.

3. Límite de visitas permitidas:
- Diagnóstico e Instalación: hasta 3 visitas.
- Instalación eléctrica certificada: hasta 2 visitas.
- Mantenimiento aire acondicionado: 1 sola visita (el gas reservado se devuelve al proveedor si falla).
- Clase y razón: cada subclase gestiona su política de riesgo e insumos.

4. Compatibilidad con cuadrillas:
- OrdenDeServicio (base): acepta cualquier cuadrilla (retorna verdadero).
- MantenimientoAireAcondicionado: exige que la cuadrilla esté certificada en refrigeración.
- Clase y razón: la orden conoce qué certificación exige para ser atendida.

5. Validación de datos de entrada:
- Cada subclase valida en su constructor sus restricciones propias (horas <= 1.5, puntos > 0, materiales >= 0, zona metropolitana y rango BTU).

---

### 3. Estados y transiciones

#### Estados de una Orden de Servicio
- Pendiente: orden registrada, en espera de asignación. Vuelve aquí si se retira de una jornada o tras visita fallida con intentos disponibles.
- En Preparación: asignada a una jornada que aún no inicia.
- En Atención: jornada en ruta atendiendo órdenes.
- Completada: servicio realizado con éxito (estado final).
- Cancelada: agotó el límite de visitas permitidas por fallos (estado final).

Transiciones de la Orden:
- Registro -> Pendiente.
- Asignar a jornada -> Pendiente a En Preparación.
- Retirar de jornada -> En Preparación a Pendiente.
- Iniciar jornada -> En Preparación a En Atención (si visitas > 0, suma recargo de Q40.00).
- Visita completada -> En Atención a Completada.
- Visita fallida con intentos restantes -> En Atención a Pendiente (suma 1 visita).
- Visita fallida sin intentos restantes -> En Atención a Cancelada (suma 1 visita).

#### Estados de una Jornada
- En Preparación: creada con cuadrilla libre; permite asignar y retirar órdenes.
- En Atención: iniciada con órdenes asignadas; no permite cambios.
- Cerrada: todas las órdenes tienen resultado calificado; libera a la cuadrilla (estado final).

Transiciones de la Jornada:
- Crear con cuadrilla libre -> En Preparación.
- Asignar/retirar órdenes -> Permanece En Preparación.
- Iniciar (con al menos 1 orden) -> En Preparación a En Atención.
- Registrar todos los resultados -> En Atención a Cerrada.

---

### 4. Clases y responsabilidades (Arquitectura MVC)

1. OrdenDeServicio (abstracta): clase base con datos comunes (código, cliente, dirección, horas estimadas, zona, estado, visitas, costo). Métodos abstractos: calcularCosto(), calcularDiasAtencion(), getMaxVisitas(), getCategoria(). Método polimórfico esCompatibleConCuadrilla(cuadrilla) que retorna true por defecto.
2. Diagnostico (extends OrdenDeServicio): valida horas estimadas <= 1.5; implementa costo fijo por zona (150/225), tiempos (1/2 días) y límite de 3 visitas.
3. Instalacion (extends OrdenDeServicio): tiene atributo propio puntos. Calcula horas por puntos, horas facturables redondeadas hacia arriba, costo por hora (Q90), tiempos (3/5 días) y límite de 3 visitas.
4. InstalacionElectricaCertificada (extends Instalacion): tiene atributo valorMateriales. Reutiliza cálculos de Instalacion mediante super: costo = super.calcularCosto() + 120 + (materiales * 0.05); tiempo = super.calcularDiasAtencion() + 2; límite de 2 visitas.
5. MantenimientoAireAcondicionado (extends OrdenDeServicio): tiene atributo capacidadBTU. Valida zona metropolitana y BTU en rango; costo base + Q110/hora + Q80 gas; tiempo de 2 días; límite de 1 visita; sobrescribe esCompatibleConCuadrilla() exigiendo certificación en refrigeración.
6. Cuadrilla: código, capacidad en horas, certificación en refrigeración y bandera de jornada activa.
7. Jornada: cuadrilla, lista de órdenes, horas asignadas, estado y registro de resultados. Valida capacidad por horas estimadas y compatibilidad de órdenes.
8. Enums: Zona (Metropolitana, Periférica), EstadoOrden (Pendiente, En Preparación, En Atención, Completada, Cancelada), EstadoJornada (En Preparación, En Atención, Cerrada), ResultadoVisita (Completada, Fallida).
9. TecniHogarController: orquestador de listas en memoria de órdenes, cuadrillas y jornadas; acumulador de ingresos y generación de reportes.
10. VistaConsola: captura datos validados desde consola, despliega menús y muestra reportes formateados a 2 decimales.
11. Main: carga los datos iniciales y arranca el ciclo de la aplicación.

#### Colaboración de constructores con super
Al instanciar una InstalacionElectricaCertificada:
1. Su constructor llama a super() hacia Instalacion enviando los datos de instalación.
2. Instalacion llama a super() hacia OrdenDeServicio enviando los datos generales (código, cliente, dirección, horas estimadas, zona).
3. OrdenDeServicio inicializa campos comunes, fija estado en Pendiente y visitas en 0.
4. Instalacion valida puntos > 0 y guarda puntos.
5. InstalacionElectricaCertificada valida materiales >= 0, guarda valorMateriales y calcula su costo total con calcularCosto().

---

### 5. Justificación de la relación entre Instalación e Instalación Eléctrica Certificada

Se definió que InstalacionElectricaCertificada hereda de Instalacion:
- Principio "Es-Un" (Liskov): una instalación eléctrica certificada es una instalación que añade certificación y garantía. Posee puntos a instalar, calcula horas por puntos (puntos x 0.75), redondea horas facturables hacia arriba y aplica la tarifa base más Q90.00 por hora.
- Reutilización (DRY): evita duplicar la fórmula de horas por puntos y redondeo de horas facturables. El costo reutiliza super.calcularCosto() y el tiempo super.calcularDiasAtencion().

#### Comparación con alternativas:
- Clases hermanas separadas (heredando ambas directo de OrdenDeServicio): ganaría independencia si las reglas cambiaran drásticamente, pero pierde al duplicar código idéntico de cálculo de puntos y horas facturables.
- Clase única con bandera booleana (Instalacion con esCertificada): reduce una clase, pero rompe polimorfismo llenando el código de if/else y forzando a instalaciones normales a tener atributos vacíos de materiales.

---

### 6. Extensibilidad ante nuevas categorías

Para agregar servicios como Paneles Solares o Cámaras de Seguridad:
- Clases a crear: solo la nueva subclase que herede de OrdenDeServicio (o Instalacion) implementando costo, tiempo, visitas y compatibilidad técnica si requiere cuadrilla especial.
- Clases a modificar: únicamente la vista y la fábrica en el controlador para añadir la opción en el menú.
- ¿Cambian las Jornadas?: NO. Jornada trabaja con OrdenDeServicio de forma polimórfica sin importar el tipo concreto.
- ¿Cambian los Reportes?: NO. Los reportes agrupan dinámicamente usando getCategoria(), getEstado() y getCostoTotal().

Cumple el Principio Abierto/Cerrado (OCP): abierto a extensión, cerrado a modificación.

---

### 7. Datos iniciales para pruebas

El sistema arranca con datos en memoria e ingresos en Q0.00:

#### Cuadrillas:
1. CUA-01: 8.0 horas, refrigeración: Sí, libre.
2. CUA-02: 6.0 horas, refrigeración: No, libre.
3. CUA-03: 4.5 horas, refrigeración: Sí, libre.

#### Órdenes (todas en estado Pendiente):
1. ORD-D01 (Diagnóstico): Carlos Ruiz, Metropolitana, 1.0 h -> Costo: Q150.00, 1 día, máx 3 visitas.
2. ORD-D02 (Diagnóstico): Ana Morales, Periférica, 1.5 h -> Costo: Q225.00, 2 días, máx 3 visitas.
3. ORD-I01 (Instalación): Inmobiliaria Miraflores, Metropolitana, 2.5 h, 5 puntos -> Horas puntos: 3.75 h, horas facturables: 4 h (horas por puntos superan horas estimadas). Costo: Q100 + (4 x Q90) = Q460.00, 3 días, máx 3 visitas.
4. ORD-I02 (Instalación): Bodegas El Trébol, Periférica, 4.0 h, 3 puntos -> Horas facturables: 4 h. Costo: Q175 + (4 x Q90) = Q535.00, 5 días, máx 3 visitas.
5. ORD-E01 (Inst. Eléctrica Certificada): Residenciales Vista Bella, Metropolitana, 2.5 h, 5 puntos, materiales Q2,000.00 -> Costo: Q460 (inst.) + Q120 (cert.) + Q100 (5% mat.) = Q680.00, 5 días, máx 2 visitas.
6. ORD-E02 (Inst. Eléctrica Certificada): Taller Industrial Sur, Periférica, 3.0 h, 4 puntos, materiales Q5,000.00 -> Costo: Q445 (inst.) + Q120 (cert.) + Q250 (5% mat.) = Q815.00, 7 días, máx 2 visitas.
7. ORD-M01 (Mantenimiento Aire Acc.): Oficinas Reforma, Metropolitana, 2.0 h, 18,000 BTU -> Costo: Q100 + (2 x Q110) + Q80 = Q400.00, 2 días, máx 1 visita.
8. ORD-M02 (Mantenimiento Aire Acc.): Banco Metropolitano, Metropolitana, 3.5 h, 36,000 BTU -> Costo: Q100 + (3.5 x Q110) + Q80 = Q565.00, 2 días, máx 1 visita.
