# Proyecto-DevOps-EP3-frontend

# Tienda de Perritos - Capa de Presentación, Interfaz Web e Inyección Proxy (Nginx)

Este repositorio aloja de manera exclusiva el código fuente, la lógica de interacción con el cliente y los manifiestos de distribución en la nube para el **Frontend** de la plataforma **Tienda de Perritos**. El componente está estructurado para operar como la cara pública de la solución, distribuyéndose elásticamente dentro de un clúster de **Amazon EKS** de alta disponibilidad.

---

## 1. Arquitectura Detallada del Componente

La interfaz de usuario funciona de manera independiente a la infraestructura del Backend, consumiendo las APIs expuestas mediante solicitudes asíncronas HTTP del lado del cliente (*Client-Side Rendering / Fetch API*).

### A. Servidor Web de Alto Rendimiento (Nginx)
* **Contenerización Avanzada:** La aplicación se empaqueta sobre una imagen ultraligera basada en **Nginx Alpine**. Nginx actúa como un servidor HTTP de alto rendimiento encargado de procesar los archivos estáticos de la interfaz (`index.html`, `app.js`, archivos multimedia).
* **Configuración del Proxy (`default.conf`):** Incluye directivas optimizadas expuestas en el puerto `80` para redireccionar de forma interna las peticiones hacia el microservicio backend y evitar problemas de políticas de seguridad en el navegador (CORS).

### B. Nodos / Fargate / Capacity Providers
El Frontend corre sobre el clúster `devopseks` utilizando **Nodos EC2 Administrados (Managed Node Groups)** en vez de soluciones Serverless como AWS Fargate.
* Los recursos del Frontend comparten el mismo clúster de cómputo elástico, optimizando el consumo de hardware disponible.
* Al usar nodos EC2 en vez de Fargate, se mitigan los tiempos de inicio diferidos (*Cold Starts*), permitiendo que la interfaz responda instantáneamente ante incrementos masivos y abruptos de visitas simultáneas en el sitio.

### C. Redes, Subredes y Security Groups (Exposición Perimetral)
La conectividad externa y el aislamiento de este componente se orquestan bajo el modelo de capas de AWS provisto por CloudFormation:
* **Subredes Públicas (Perímetro):** El Service de Kubernetes está configurado como tipo `LoadBalancer`. Al procesarlo, AWS aprovisiona de forma nativa un **Application Load Balancer (ALB)** público. Este balanceador de carga se ubica en las subredes públicas para recibir el tráfico web directo desde Internet (Puerto 80).
* **Subredes Privadas (Cómputo):** Los Pods reales del Frontend que contienen a Nginx se ejecutan de manera segura dentro de las subredes privadas. El LoadBalancer recibe la petición del cliente en la subred pública y la redirige hacia los Pods en la subred privada de forma segura.
* **Security Groups (Cortafuegos):** El Grupo de Seguridad del LoadBalancer está abierto al mundo en el puerto 80 (`0.0.0.0/0`), pero el Grupo de Seguridad de los Nodos del Frontend está restringido: **solo acepta peticiones HTTP si provienen del LoadBalancer**, bloqueando accesos directos maliciosos a los servidores.

### D. Elasticidad y Disponibilidad Distribuida
* **Distribución Geográfica:** Los Pods son asignados dinámicamente a lo largo de las distintas zonas de disponibilidad físicas que componen el grupo de nodos de AWS Academy.
* **Autoescalado Horizontal:** Cuenta con un manifiesto **Horizontal Pod Autoscaler (HPA)** dedicado, asegurando la adaptabilidad automática de los recursos en escenarios de alta concurrencia masiva (como simulacros de eventos CyberDay), ajustando los contenedores disponibles basándose en el estado de carga actual de los nodos de cómputo.

---

## 2. Estructura Completa del Proyecto

```text
├── .github/
│   └── workflows/
│       └── deployment.yml       # Pipeline automatizado de CI/CD (GitHub Actions)
├── k8s/
│   ├── frontend-deployment.yaml # Enlace al archivo del frontend-deployment
│   ├── frontend-service.yaml    # Service tipo LoadBalancer para la salida a internet
│   └── frontend-hpa.yaml        # Configuración elástica de autoescalado horizontal
├── frontend/
│   ├── Dockerfile                   # Configuración del empaquetado basado en Nginx Alpine
│   ├── default.conf                 # Archivo de configuración del servidor Nginx
│   ├── index.html                   # Interfaz visual y maquetación de la tienda
│   ├── app.js                       # Lógica JavaScript (Consumo asíncrono de la API Backend)
└── README.md                    # Documentación técnica principal del repositorio
