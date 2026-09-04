Feature Specification: Calcular Penalización 

Created: 2026-09-03 

 

User Scenarios & Testing  

User Story 1 - Sanción progresiva por retraso en check-out (Priority: P1) 

Como sistema, cuando un estudiante realiza el check-out de un equipo y excede el tiempo límite de uso, debo contar sus infracciones previas en los últimos 30 días y aplicar la sanción correspondiente según la escala progresiva configurada (1 día para primera infracción, 2 días para segunda, etc.), restando siempre 5 puntos del score de confianza por cada infracción. 

Why this priority: El retraso en la entrega de equipos es la infracción más común y tiene mayor impacto en la disponibilidad de recursos. La sanción progresiva es el corazón del módulo de sanciones y debe funcionar siempre desde el MVP. 

Independent Test: Puede ser probado independientemente simulando un estudiante con 0 infracciones previas que realiza check-out con 30 minutos de retraso, y verificando que el sistema aplica 1 día de suspensión y resta 5 puntos del score de confianza (de 50 a 45). 

Acceptance Scenarios: 

Scenario: Primera infracción por retraso (sin antecedentes) 

Given Un estudiante con score de confianza inicial de 50 puntos y 0 infracciones en los últimos 30 días 

When Realiza check-out con 30 minutos de retraso (supera el umbral de 15 minutos) 

Then El sistema aplica 1 día de suspensión, resta 5 puntos del score (queda en 45), y registra la infracción en el historial 

Scenario: Segunda infracción por retraso (reincidencia) 

Given Un estudiante con 1 infracción previa en los últimos 30 días y score actual de 45 puntos 

When Realiza check-out con 20 minutos de retraso 

Then El sistema aplica 2 días de suspensión, resta 5 puntos del score (queda en 40), y registra la infracción 

Scenario: Quinta infracción por retraso (sanción máxima) 

Given Un estudiante con 4 infracciones previas en los últimos 30 días y score actual de 30 puntos 

When Realiza check-out con 45 minutos de retraso 

Then El sistema aplica 7 días de suspensión, resta 5 puntos del score (queda en 25), y registra la infracción 

Scenario: Infracción que no supera el umbral mínimo 

Given Un estudiante con 0 infracciones previas 

When Realiza check-out con 10 minutos de retraso (umbral mínimo configurado en 15 minutos) 

Then El sistema NO calcula penalización porque el retraso no supera el umbral mínimo 

 

User Story 2 - Sanción progresiva por inasistencia (Priority: P1) 

Como sistema, cuando un profesor o administrador reporta que un estudiante no asistió a su reserva, debo contar sus infracciones previas en los últimos 30 días y aplicar la sanción progresiva correspondiente (1 día, 2 días, etc.) de forma obligatoria, restando siempre 5 puntos del score de confianza por cada infracción. 

Why this priority: La inasistencia es una falta grave que afecta la disponibilidad de recursos y la planificación. Es igual de crítica que el retraso y debe estar disponible desde el MVP. 

Independent Test: Puede ser probado independientemente simulando el reporte de inasistencia para un estudiante con 2 infracciones previas y verificando que el sistema aplica 3 días de suspensión y resta 5 puntos del score (de 40 a 35). 

Acceptance Scenarios: 

Scenario: Primera inasistencia 

Given Un estudiante con 0 infracciones previas y score de 50 puntos 

When El profesor ejecuta el flujo de Reportar no asistencia 

Then El sistema aplica obligatoriamente 1 día de suspensión, resta 5 puntos del score (queda en 45), y registra la infracción 

Scenario: Tercera inasistencia (reincidencia grave) 

Given Un estudiante con 2 infracciones previas en los últimos 30 días y score de 40 puntos 

When El profesor reporta una nueva inasistencia 

Then El sistema aplica 3 días de suspensión, resta 5 puntos del score (queda en 35), y registra la infracción 

 

Scenario: Más de 5 inasistencias (sanción máxima) 

Given Un estudiante con 5 infracciones previas en los últimos 30 días y score de 25 puntos 

When El profesor reporta una nueva inasistencia 

Then El sistema aplica 30 días de suspensión, resta 5 puntos del score (queda en 20), y registra la infracción 

 

User Story 3 - Gestión de score de confianza (Priority: P2) 

Como sistema, debo mantener el score de confianza de cada estudiante iniciando en 50 puntos y actualizarlo automáticamente después de cada sanción, restando siempre 5 puntos. El score siempre se mueve en incrementos/decrementos de 5 puntos. 

Why this priority: El score de confianza es un mecanismo de control adicional que permite identificar estudiantes con mal comportamiento recurrente. Es importante pero no bloqueante para el cálculo base de sanciones. 

Independent Test: Puede ser probado independientemente creando un estudiante nuevo con score inicial de 50, aplicando 3 sanciones, y verificando que el score final es 35 (reducciones de 5 puntos cada una). 

Acceptance Scenarios: 

Scenario: Score inicial correcto 

Given Un estudiante nuevo es registrado en el sistema 

When Se consulta su score de confianza 

Then El score debe ser 50 puntos 

Scenario: Reducción progresiva del score (siempre en 5) 

Given Un estudiante con score de 50 puntos y 0 infracciones 

When Recibe 3 sanciones consecutivas (3 infracciones en 30 días) 

Then El score se reduce a 35 puntos (50 - 5 - 5 - 5) 

Scenario: Score en 0 (bloqueo automático) 

Given Un estudiante con score de 5 puntos y 9 infracciones previas 

When Recibe una nueva sanción que resta 5 puntos 

Then El score queda en 0 y el sistema bloquea automáticamente al estudiante (NO puede realizar nuevos préstamos) 

Scenario: Recuperación de puntos (buen comportamiento) 

Given Un estudiante con score de 30 puntos que ha completado 30 días sin infracciones 

When El administrador ejecuta el flujo de Recuperar score de confianza 

Then El sistema aumenta 5 puntos del score (queda en 35), respetando siempre el incremento en múltiplos de 5 

 

User Story 4 - Configuración de escalas progresivas (Priority: P3) 

Como administrador, necesito poder modificar los rangos de la escala progresiva (días de suspensión) a través de una interfaz de administración, sin necesidad de modificar el código fuente. Los puntos del score son fijos (siempre 5 puntos por infracción) y no son configurables. 

Why this priority: Esta funcionalidad es importante para la mantenibilidad del sistema pero no es crítica para el MVP, ya que se puede comenzar con valores predefinidos y ajustarlos después. 

Independent Test: Puede ser probado independientemente accediendo a la interfaz de administración, modificando la escala (ej. cambiar "5 infracciones = 7 días" a "5 infracciones = 10 días"), y verificando que las siguientes sanciones usan el nuevo valor. El score siempre resta 5 puntos, sin importar la modificación. 

Acceptance Scenarios: 

Scenario: Modificación de días de suspensión 

Given Un administrador accede a la configuración de escalas 

When Cambia la escala para 2 infracciones previas de 3 días a 5 días 

Then El sistema guarda el cambio y las siguientes sanciones con 2 infracciones previas aplican 5 días. El score sigue restando 5 puntos. 

Scenario: Adición de nuevo rango en la escala 

Given Un administrador accede a la configuración de escalas 

When Agrega un nuevo rango "6 infracciones = 45 días" 

Then El sistema guarda el nuevo rango y lo aplica cuando un estudiante alcanza 6 infracciones. El score sigue restando 5 puntos. 

 

 

 

 

User Story 5 - Notificación de sanciones al estudiante (Priority: P2) 

Como sistema, después de calcular una penalización, debo notificar al estudiante a través del Módulo 1 o 2, informándole el motivo, los días de suspensión, los 5 puntos restados del score de confianza y su nuevo score actualizado. 

Why this priority: La notificación es esencial para que el estudiante conozca su situación y pueda tomar acciones correctivas. Es importante pero puede implementarse después del cálculo base. 

Independent Test: Puede ser probado independientemente calculando una sanción y verificando que el estudiante recibe una notificación con todos los detalles de la sanción aplicada. 

Acceptance Scenarios: 

Scenario: Notificación de primera infracción 

Given Un estudiante recibe su primera sanción por retraso (1 día, -5 puntos) 

When El sistema calcula la penalización 

Then El estudiante recibe una notificación con: "Motivo: Retraso en entrega. Sanción: 1 día de suspensión. Score de confianza: 45 puntos (-5)" 

Scenario: Notificación de sanción grave 

Given Un estudiante recibe su quinta sanción (7 días, -5 puntos) 

When El sistema calcula la penalización 

Then El estudiante recibe una notificación con: "Motivo: Retraso en entrega. Sanción: 7 días de suspensión. Score de confianza: 25 puntos (-5). Esta es su quinta infracción en 30 días." 

 

Edge Cases 

¿Qué sucede cuando un estudiante tiene infracciones de diferentes tipos (ej. 2 retrasos y 1 inasistencia)? ¿Se cuentan por separado o se acumulan? 

Respuesta: Se acumulan todas las infracciones en los últimos 30 días, independientemente del tipo. Cada infracción siempre resta 5 puntos. 

¿Cómo se maneja cuando el score de confianza llega a 0? 

Respuesta: El estudiante queda automáticamente bloqueado y no puede realizar nuevos préstamos hasta que un administrador lo habilite o el sistema permita recuperación de puntos. 

 

¿Qué pasa si el estudiante tiene 6 infracciones pero la escala solo llega hasta 5? 

Respuesta: El sistema aplica el último rango definido (el más alto) para cualquier número de infracciones superior al último rango configurado. 

¿Cómo se reinicia el contador de infracciones? 

Respuesta: Automáticamente después de 30 días sin infracciones. El período es configurable por el administrador. 

¿Qué sucede si el estudiante está bloqueado por daño técnico y además tiene una penalización por retraso? 

Respuesta: Ambas sanciones coexisten y el bloqueo prevalece. El estudiante no puede realizar préstamos hasta resolver ambas situaciones. 

¿Cómo se maneja cuando el score de confianza no es múltiplo de 5? 

Respuesta: El sistema siempre mantiene el score en múltiplos de 5, ya que solo se incrementa o decrementa en 5 puntos. Nunca debería haber un score que no sea múltiplo de 5. 

¿El score siempre se decrementa en 5 o puede variar? 

Respuesta: El score siempre se decrementa en 5 puntos por cada infracción. Este valor es fijo y no configurable. Solo los días de suspensión son configurables en la escala progresiva. 

 

Requirements  

Functional Requirements 

FR-001: El sistema debe calcular una penalización cuando el flujo de Realizar check-out indique que el tiempo de uso superó el límite establecido (umbral_minimo configurado en Sancion_Reglas), aplicando la escala progresiva definida en Sancion_Escala_Progresiva basada en el historial de infracciones previas del estudiante en los últimos 30 días. 

FR-002: El sistema debe calcular obligatoriamente una penalización siempre que se ejecute el flujo de Reportar no asistencia, aplicando la escala progresiva definida en Sancion_Escala_Progresiva basada en el historial de infracciones previas del estudiante en los últimos 30 días. 

FR-003: El sistema debe determinar la gravedad de la sanción consultando la tabla Sancion_Escala_Progresiva, donde cada rango de infracciones_previas tiene asociado un número de días de suspensión. Los puntos del score de confianza siempre son fijos: -5 por cada infracción. 

FR-004: El sistema debe vincular la penalización generada con el perfil del estudiante infractor, almacenando en su historial la infracción, los días de suspensión aplicados, los 5 puntos restados del score de confianza y el nuevo score resultante. 

FR-005: El sistema debe registrar el motivo exacto de la penalización (Retraso en entrega, Inasistencia, etc.), tomando el valor del campo motivo desde la tabla Sancion_Reglas. 

FR-006: El sistema debe disparar condicionalmente («extend») el flujo de Actualizar score de confianza cuando la tabla Sancion_Reglas indique afecta_score_confianza = TRUE, restando siempre 5 puntos del score actual del estudiante. 

FR-007: El sistema debe notificar al estudiante, a través del Módulo 1 o 2, la penalización impuesta incluyendo: motivo, días de suspensión, 5 puntos restados del score de confianza y su nuevo score actualizado. 

FR-008: El sistema debe mantener un score de confianza inicial de 50 puntos para cada estudiante, que siempre se incrementará o disminuirá en múltiplos de 5 según el comportamiento. 

FR-009: El sistema debe contar las infracciones previas del estudiante considerando solo los últimos 30 días (período configurable) para determinar la escala aplicable. 

FR-010: El sistema debe permitir a los administradores gestionar (crear, modificar, eliminar) los rangos de la escala progresiva en la tabla Sancion_Escala_Progresiva a través del Módulo 3, permitiendo ajustar únicamente los días de suspensión. El valor de -5 puntos por infracción es fijo y no configurable. 

FR-011: El sistema debe aplicar la penalización más alta cuando el número de infracciones previas supere el último rango definido en Sancion_Escala_Progresiva. 

FR-012: El sistema debe registrar en el historial de sanciones el estado de la penalización (Pendiente, Cumplida, Apelada) para permitir su seguimiento y gestión. 

FR-013: El sistema debe bloquear automáticamente al estudiante (impidiendo nuevos préstamos) cuando su score de confianza alcance 0 puntos. 

FR-014: El sistema debe permitir a los administradores configurar el período de tiempo (días) para contar infracciones previas (valor por defecto: 30 días). 

FR-015: El sistema debe acumular todas las infracciones del estudiante (retrasos e inasistencias) en el período configurado para determinar la escala aplicable. 

FR-016: El sistema debe garantizar que el score de confianza siempre sea un múltiplo de 5, ya que solo se modifica en incrementos/decrementos de 5 puntos. 

FR-017: El sistema debe permitir a los administradores ejecutar un flujo de Recuperar score de confianza que aumente 5 puntos al estudiante por cada período de buen comportamiento (ej. 30 días sin infracciones). 

Key Entities  

Sancion_Reglas: Entidad que contiene la configuración base para cada tipo de penalización. Incluye motivo, unidad de cálculo, umbral mínimo, indicador de afectación al score de confianza y estado activo/inactivo. Define el "qué" y el "cuándo" de la sanción. 

Sancion_Escala_Progresiva: Entidad que define la progresión de sanciones según el historial de infracciones del estudiante. Cada fila asocia un número de infracciones previas con días de suspensión. Los puntos del score son fijos (-5 por infracción) y no se almacenan en esta tabla. Permite aplicar sanciones crecientes (1 día, 2 días, 1 semana, 1 mes) según reincidencia. 

Historial_Sanciones: Entidad que registra cada penalización aplicada. Incluye referencia a la regla y escala aplicada, identificación del estudiante, fecha de aplicación, días de suspensión, 5 puntos restados del score, score resultante, estado de la sanción (Pendiente, Cumplida, Apelada) y motivo. Sirve como trazabilidad completa de todas las sanciones. 

Score_Confianza: Entidad que almacena el nivel de confianza del estudiante. Inicia en 50 puntos y siempre se actualiza en múltiplos de 5 (restando 5 por cada infracción o sumando 5 por recuperación de puntos). Permite identificar patrones de conducta y disparar bloqueos automáticos cuando alcanza 0 puntos. 

Estudiante: Entidad que representa al infractor. Contiene su información personal (nombre, ID, correo, etc.), su score de confianza actual (siempre múltiplo de 5), su historial completo de sanciones y su estado (Activo, Bloqueado por score, Bloqueado por daño, etc.). 

Configuracion_Sistema: Entidad que almacena parámetros globales del sistema, incluyendo el período de tiempo (días) para contar infracciones previas (valor por defecto: 30 días). 

 

Success Criteria  

Measurable Outcomes 

SC-001: El sistema debe calcular cualquier penalización en menos de 2 segundos desde que se dispara el flujo (check-out o reporte de inasistencia), incluyendo la consulta al historial de infracciones, la búsqueda en la escala progresiva y la actualización del score de confianza restando 5 puntos. 

SC-002: El score de confianza debe iniciar en 50 puntos para todos los estudiantes nuevos y actualizarse correctamente después de cada sanción, restando exactamente 5 puntos por infracción, con una precisión del 100% en los cálculos. 

SC-003: El score de confianza debe mantenerse siempre como un múltiplo de 5 en todo momento, garantizando que nunca existan valores como 47, 38 o 22. 

SC-004: El 100% de las sanciones deben aplicar correctamente la escala progresiva basada en el conteo de infracciones previas en el período configurado (30 días por defecto). 

SC-005: Los administradores deben poder modificar los rangos de la escala progresiva (días de suspensión) en menos de 5 minutos, viendo los cambios reflejados en la siguiente sanción calculada, sin necesidad de reiniciar el sistema. Los -5 puntos del score permanecen fijos. 

SC-006: El sistema debe notificar al estudiante en menos de 5 segundos después de calcular la penalización, con todos los detalles de la sanción aplicada incluyendo los 5 puntos restados. 

SC-007: El sistema debe reducir en un 80% las sanciones desproporcionadas gracias al enfoque progresivo y parametrizable, comparado con un sistema de sanciones fijas. 

SC-008: El sistema debe bloquear automáticamente al estudiante cuando su score de confianza alcance 0 puntos, con una efectividad del 100% (ningún estudiante con score 0 puede realizar nuevos préstamos). 

SC-009: El tiempo de cambio de políticas de sanciones (días de suspensión) debe reducirse de 2 semanas (despliegue de código) a 5 minutos (actualización en interfaz de administración). 

SC-010: El historial de sanciones debe permitir auditoría completa, registrando el 100% de las infracciones con todos los campos obligatorios (motivo, días de suspensión, 5 puntos restados, score resultante, fecha, estado). 
