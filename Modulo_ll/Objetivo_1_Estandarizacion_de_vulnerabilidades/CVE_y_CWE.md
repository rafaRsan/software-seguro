# Estandarización de Vulnerabilidades (CVE y CWE)

## Definición de CVE y CWE
*   **CVE (Common Vulnerabilities and Exposures):** Es un sistema que proporciona un identificador 
único y estandarizado (por ejemplo, CVE-2021-44228) para vulnerabilidades de seguridad informáticas
conocidas y divulgadas públicamente. Actúa como un diccionario global para que toda la comunidad
se refiera exactamente al mismo fallo de seguridad en un producto específico.
*   **CWE (Common Weakness Enumeration):** Es un catálogo formal que clasifica los tipos de 
debilidades de software y hardware que pueden provocar vulnerabilidades. No describe un fallo en un 
sistema en particular, sino la causa raíz técnica subyacente (por ejemplo, CWE-79 hace referencia a 
Cross-Site Scripting, y CWE-89 a Inyección SQL).

## Diferencia 
La diferencia esta en lo específico frente a lo general. El **CWE describe la categoría o tipo de debilidad** 
arquitectónica o de programación, mientras que el **CVE identifica una falla específica descubierta en un software real** . 
Ejemplo: Un desarrollador cometió un error de tipo **CWE-89** (Inyección SQL) al programar WordPress.
Cuando un investigador descubre cómo explotar eso en la versión 6.0 de WordPress, se le asigna un 
**CVE específico** (ej. CVE-202X-XXXX) a ese hallazgo.

## Asignación y Consulta
*   **CNAs (CVE Numbering Authorities):** Son las organizaciones 
(como Microsoft, Apple, Google, o investigadores independientes autorizados) delegadas por el 
programa CVE (gestionado por MITRE) para asignar los identificadores CVE a las vulnerabilidades 
descubiertas dentro de sus productos o ámbitos de acción.
*   **Bases de consulta pública:** Los profesionales de ciberseguridad consultan estos 
identificadores principalmente en dos bases de datos globales:
    *   El sitio oficial del catálogo de **MITRE**.
    *   La **NVD (National Vulnerability Database)**, mantenida por el NIST del gobierno de Estados Unidos,
      que además añade métricas como el puntaje CVSS.

## Herramientas Automatizadas de la Industria
Para detectar vulnerabilidades que ya tienen un CVE asignado en una infraestructura, 
la industria utiliza escáneres automatizados. Dos ejemplos destacados son:
1.  **Nessus:** Uno de los escáneres de vulnerabilidades comerciales más completos y utilizados en 
auditorías corporativas.
2.  **Nuclei:** Una herramienta de código abierto muy rápida y basada en plantillas (templates) 
YAML, sumamente utilizada en pentesting moderno para detectar CVEs recientes en aplicaciones web.
(Otra alternativa válida es **OpenVAS**).
