# Práctica de Peritaje Informático
Auditoría Informática – Licenciatura en Sistemas de Información – FCEQyN – UNaM

Entorno de trabajo para la práctica de informática forense. Contiene la evidencia de un **caso ficticio** y las herramientas para analizarlo, todo dentro de un contenedor Docker. No hace falta instalar nada más.

**La consigna está en [GUIA_PRACTICA.md](GUIA_PRACTICA.md).**

## Preparación (hacerla ANTES de la clase)

1. Instalar Docker:
   - Windows / macOS: [Docker Desktop](https://www.docker.com/products/docker-desktop/). En Windows, con el backend WSL2.
   - Linux: Docker Engine y el plugin `docker compose`.
2. Clonar este repositorio y descargar la imagen (unos 200/300 MB):
   ```bash
   git clone https://github.com/tinxo/auditoria-peritaje.git
   cd auditoria-peritaje
   docker compose pull
   ```
3. Probar que funciona:
   ```bash
   docker compose run --rm perito
   ```
   Tienen que ver el mensaje de bienvenida y el prompt `perito@lab:/casos$`. Salen con `exit`.

Si no pudieron hacer la preparación, avisen al comienzo de la clase: en el aula va a haber una copia de la imagen para cargar sin conexión.

## Contenido

| Ruta | Qué es |
|---|---|
| `GUIA_PRACTICA.md` | Caso, consignas por etapa y qué se entrega |
| `plantillas/` | Acta de adquisición, cadena de custodia e informe pericial (se copian a `casos/` y se completan ahí) |
| `casos/` | Su carpeta de trabajo. Es lo único que el contenedor puede escribir |
| `docker-compose.yml` | Configuración del contenedor: solo lectura y sin red |

## Problemas frecuentes

- **`permission denied ... docker.sock`** (Linux): agreguen su usuario al grupo `docker` (`sudo usermod -aG docker $USER`) y vuelvan a iniciar sesión.
- **`Cannot connect to the Docker daemon`**: Docker Desktop no está abierto.
- **No aparecen archivos en `casos/`** (Windows): cloné el repositorio dentro de su carpeta de usuario y no en una carpeta de OneDrive.

Todos los datos del caso son sintéticos. Cualquier parecido con personas o empresas reales es coincidencia.
