# Práctica 3: Agente con Microsoft Agents Framework + Microsoft Foundry (Azure OpenAI) y herramienta de Web Search

## Objetivo

En esta práctica vamos a:

1. Reutilizar el entorno de la Práctica 2
2. Instalar Azure CLI (`az`) y Azure Developer CLI (`azd`)
3. Autenticarnos contra Azure mediante *device-code flow* (compatible con SSH/headless)
4. Identificar los datos necesarios del recurso de **Microsoft Foundry / Azure OpenAI** y rellenar el archivo `.env`
5. Asignarnos el rol de plano de datos sobre el recurso
6. Ejecutar un agente conectado a un modelo en Foundry (`gpt-5.4-mini`) con:
   - Una **herramienta local** (`get_enrollment_deadline_info`) idéntica a la Práctica 2
   - Una **herramienta nativa de búsqueda web** (`get_web_search_tool()`) del Agent Framework
   - **Bucle interactivo** de diálogo

> Nota: Aquí el modelo vive en Azure. Lo único que cambia respecto al código es el cliente y la autenticación; el patrón del `Agent` y de las herramientas es el mismo.

---

# 1. Punto de partida

Partimos de la VM Ubuntu de la Práctica 2 (recomendado 2vCPUs, 16GB RAM — `F2amds_v7`) con:

- `uv` instalado
- Carpeta `msft-agents/` ya creada
- Entorno virtual `.msft` con `agent-framework-core`, `agent-framework-openai`, `agent-framework-foundry`, `azure-ai-projects`, `azure-identity`, `python-dotenv` y `rich`

Activamos el entorno:

```bash
cd msft-agents
source .msft/bin/activate
```


---

# 2. Instalar Azure CLI (`az`) 

El cliente `az` es nuestra herramienta principal para inspeccionar recursos y gestionar asignaciones de rol.

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az version
```

---

# 3. Instalar Azure Developer CLI (`azd`)

`azd` mantiene una caché de tokens **independiente** de `az`. El Agent Framework con `AzureCliCredential` usa la caché de `az`, pero conviene tener `azd` instalado para escenarios de despliegue.

```bash
curl -fsSL https://aka.ms/install-azd.sh | bash
azd version
```

---

# 4. Autenticarse con Azure (device-code flow)

Si estamos en una VM por SSH no hay navegador, así que usamos *device code*. Esto nos da una URL y un código que pegamos en el navegador de nuestro portátil.

## 4.1. Login con `az`


```bash
az login --use-device-code --tenant <AZURE_TENANT_ID>
```

Sustituye `<AZURE_TENANT_ID>` por el tuyo.

Salida esperada:

```text
To sign in, use a web browser to open the page https://microsoft.com/devicelogin
and enter the code XXXXXXXXX to authenticate.
```

Abrimos la URL en el portátil, pegamos el código, completamos el login con la cuenta corporativa, y volvemos al terminal.

## 4.2. Fijar la suscripción activa

```bash
az account set --subscription <AZURE_SUBSCRIPTION_ID>
az account show -o table
```

## 4.3. Login con `azd` (recomendado)

```bash
azd auth login --tenant-id <AZURE_TENANT_ID> --use-device-code
```

---

# 5. Identificar los datos del recurso de Foundry

Necesitamos cuatro valores para nuestro `.env`. Vamos a obtenerlos por CLI.

## 5.1. Tenant ID (ya lo tenemos)

```bash
az account show --query tenantId -o tsv
```

## 5.2. Endpoint del recurso de Foundry / Azure OpenAI

Listamos los recursos `cognitiveservices` de tipo `AIServices` en el grupo de recursos creado:

```bash
az group list --query "[].{name:name, location:location}" -o table
```

```bash
az cognitiveservices account list \
  --resource-group <RESOURCE_GROUP> \
  --query "[].{name:name, endpoint:properties.endpoint, kind:kind}" \
  -o table
```

Apuntamos el `endpoint`. Tendrá una forma como:

```text
https://<resource-name>.services.ai.azure.com
```

## 5.3. Nombre del deployment del modelo

Obtengamos todos los Recursos primero

```bash
az cognitiveservices account list \
  --resource-group <RESOURCE_GROUP> \
  --query "[].{name:name, kind:kind, location:location}" \
  -o table
```
después:

```bash
az cognitiveservices account deployment list \
  --name <RESOURCE_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --query "[].{name:name, model:properties.model.name, version:properties.model.version}" \
  -o table
```

Apuntamos el `name` del deployment (en nuestro caso `gpt-5.4-mini`).

## 5.4. Project endpoint de Foundry

Es el endpoint del recurso seguido de `/api/projects/<project-name>`:

```text
https://<resource-name>.services.ai.azure.com/api/projects/<project-name>
```

---

# 6. Crear el archivo `.env`

Dentro de `msft-agents/`:

```bash
nano .env
```

Contenido (rellena con tus valores):

```env
AZURE_TENANT_ID=<your-tenant-guid>
AZURE_OPENAI_ENDPOINT=https://<your-resource-name>.services.ai.azure.com
AZURE_AI_MODEL_DEPLOYMENT_NAME=gpt-5.4-mini
FOUNDRY_PROJECT_ENDPOINT=https://<your-resource-name>.services.ai.azure.com/api/projects/<your-project-name>
```

Guarda y sal.

> El Agent Framework **no** lee `.env` automáticamente. Lo cargaremos en el script con `python-dotenv`.

---

# 7. Asignarnos el rol de plano de datos

Para invocar el modelo con AAD necesitamos el rol **`Cognitive Services OpenAI User`** sobre el recurso. Sin él, el token será válido pero las llamadas devolverán `401`.

## 7.1. Obtener el ID del recurso y el object ID del usuario

```bash
RESOURCE_ID=$(az cognitiveservices account show \
  --name <RESOURCE_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --query id -o tsv)

USER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)

echo $USER_OBJECT_ID

```

## 7.2. Crear la asignación de rol

```bash
az role assignment create \
  --role "Cognitive Services OpenAI User" \
  --assignee-object-id "$USER_OBJECT_ID" \
  --assignee-principal-type User \
  --scope "$RESOURCE_ID"
```

> La propagación tarda **1 – 3 minutos**. Paciencia.

## 7.3. Verificar la asignación

```bash
az role assignment list \
  --assignee-object-id "$USER_OBJECT_ID" \
  --scope "$RESOURCE_ID" \
  -o table
```

---

# 8. Test del token AAD

Antes de ejecutar el agente, verificamos que la cadena de autenticación funciona:

```bash
azd auth token \
  --scope "https://cognitiveservices.azure.com/.default" \
  --tenant-id <AZURE_TENANT_ID> \
  --output json
```

Si devuelve un JSON con `token` y `expiresOn`, todo está en orden.


---

# 10. Crear el script del agente

Crear archivo:

```bash
nano foundry_model.py
```

Contenido:

```python


"""
Práctica 3: Agente conectado a Microsoft Foundry (Azure OpenAI) con dos herramientas:
  - get_enrollment_deadline_info: herramienta local (misma que en Práctica 2)
  - get_web_search_tool: herramienta nativa del Agent Framework para
    búsquedas web en tiempo real, vía el Responses API.

Prerequisitos:
  - az login completado y rol "Cognitive Services OpenAI User" asignado
  - .env con AZURE_OPENAI_ENDPOINT y AZURE_AI_MODEL_DEPLOYMENT_NAME

Ejecutar:
  python foundry_model.py
"""

import asyncio
import logging
import os
from datetime import date

from agent_framework import Agent, tool
from agent_framework.openai import OpenAIChatClient
from azure.identity import AzureCliCredential
from dotenv import load_dotenv
from rich.console import Console
from rich.logging import RichHandler
from rich.markdown import Markdown

load_dotenv(override=True)

console = Console()
logger = logging.getLogger("stage1")


@tool
def get_enrollment_deadline_info() -> dict:
    """Return enrollment timeline details for health insurance plans."""

    logger.info("[tool] get_enrollment_deadline_info()")

    return {
        "enrollment_opens": "2026-11-11",
        "enrollment_closes": "2026-11-30",
    }


# --- Cliente de Foundry (Azure OpenAI) vía Responses API ---
#
# Usamos OpenAIChatClient con azure_endpoint para enrutar contra Azure.
# La Responses API es la que habilita las "hosted tools" como web search.
client = OpenAIChatClient(
    model=os.environ["AZURE_AI_MODEL_DEPLOYMENT_NAME"],
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_version=os.getenv("AZURE_OPENAI_API_VERSION", "preview"),
    credential=AzureCliCredential(),
)

# --- Herramienta de web search nativa del Agent Framework ---
#
# El cliente expone get_web_search_tool() para construir la tool
# pre-configurada con contexto de localización del usuario.
web_search_tool = client.get_web_search_tool(
    user_location={"city": "Madrid", "region": "ES"},
)

agent = Agent(
    client=client,
    instructions=(
        f"You are an internal HR helper. Today's date is {date.today().isoformat()}. "
        "Use the available tools to answer questions:\n"
        " - For benefits enrollment timing, call get_enrollment_deadline_info.\n"
        " - For anything that requires fresh, real-world information, "
        "use the web search tool and cite the source.\n"
        "Always ground your answers in tool results."
    ),
    tools=[get_enrollment_deadline_info, web_search_tool],
)


async def main():
    console.print("[bold green]Interactive HR Agent (Foundry + Web Search)[/bold green]")
    console.print("Type /q to quit.\n")

    history = []

    while True:

        user_input = console.input(
            "[bold cyan]You > [/bold cyan]"
        ).strip()

        if not user_input:
            continue

        if user_input.lower() in {"/q", "q", "quit", "exit"}:
            console.print("\nGoodbye!")
            break

        history.append(user_input)

        response = await agent.run(history)

        history.append(response.text)

        console.print("\n[bold magenta]Agent >[/bold magenta]")
        console.print(Markdown(response.text))
        console.print()


if __name__ == "__main__":

    logging.basicConfig(
        level=logging.INFO,
        format="%(message)s",
        handlers=[
            RichHandler(
                console=console,
                show_path=False
            )
        ],
    )

    # Bajamos el ruido de azure-identity en consola
    logging.getLogger("azure.identity").setLevel(logging.WARNING)
    logging.getLogger("azure.core").setLevel(logging.WARNING)

    asyncio.run(main())


```

---

# 11. Ejecutar el agente

```bash
python foundry_model.py
```

---

# 12. Probar el agente

## 12.1. Pregunta sobre la herramienta local

```text
When does benefits enrollment open?
```

Salida esperada (el agente invoca `get_enrollment_deadline_info`):

```text
Benefits enrollment opens on 2026-11-11.
```

## 12.2. Pregunta que requiere web search

```text
noticias recientes sobre .....   ?
```

El agente invocará la herramienta de web search, recuperará información reciente y devolverá un resumen.


---

# 13. Salir del chat

```text
/q
```

---

# Estructura final del proyecto

```text
msft-agents/
│
├── .msft/
├── .env
├── requirements.txt
├── setup_msft_agent_env.sh
├── local_model.py          # Práctica 2 (Ollama)
└── foundry_model.py        # Práctica 3 (Foundry + web search)
```

---

# Comandos útiles

## Ver la cuenta activa

```bash
az account show -o table
```

## Listar deployments del recurso

```bash
az cognitiveservices account deployment list \
  --name <RESOURCE_NAME> \
  --resource-group <RESOURCE_GROUP> \
  -o table
```

## Refrescar el token de `az` si caduca

```bash
az login --use-device-code --tenant <AZURE_TENANT_ID>
```

## Refrescar el token de `azd` si caduca

```bash
azd auth login --tenant-id <AZURE_TENANT_ID> --use-device-code
```

## Desactivar entorno virtual

```bash
deactivate
```

---
