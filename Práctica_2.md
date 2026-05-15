# Práctica 2: Agente local con Microsoft Agents Framework y Ollama en Ubuntu

## Objetivo

En esta práctica vamos a:

1. Instalar Ollama en Ubuntu ( recomendado 2vCPUs, 16GB RAM - F2amds_v7 )
2. Instalar `uv`
3. Crear un entorno virtual `.msft`
4. Instalar dependencias desde `requirements.txt`
5. Descargar un modelo local
6. Ejecutar un agente local usando Microsoft Agents Framework + Ollama

---

# 1. Actualizar el sistema

```bash
sudo apt update
```

---

# 2. Instalar Ollama

## Descargar e instalar

```bash
curl -fsSL https://ollama.com/install.sh | sh

ollama --version
```

---

# 3. Iniciar el servicio de Ollama (Normalmente arranca solo)

Podemos lanzarlo manualmente sino lo vemos en los procesos:

```bash
ps -aux | grep "ollama"

nohup ollama serve > /tmp/ollama.log 2>&1 &
```

Esperamos unos segundos


---

# 4. Verificar que la API responde

```bash
curl http://localhost:11434/
```

Si todo funciona correctamente veremos un JSON como respuesta.

---

# 5. Instalar uv

## Descargar e instalar

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh

export PATH="$HOME/.local/bin:$PATH"
```

## Verificar instalación

```bash
uv --version
```

---

# 6. Crear el proyecto

## Crear carpeta

```bash
mkdir msft-agents
cd msft-agents
```

---

# 7. Crear entorno virtual `.msft`

```bash
uv venv .msft -p 3.13
```

---

# 8. Activar el entorno virtual

```bash

bash

source .msft/bin/activate
```

El prompt debería mostrar algo similar a:

```text
(.msft)
```

---

# 9. Crear archivo requirements.txt

Crear el archivo:

```bash
nano requirements.txt
```

Contenido:

```txt
azure-identity>=1.25.3
python-dotenv>=1.2.2
agent-framework-core>=1.0.1
agent-framework-openai>=1.0.1
agent-framework-foundry>=1.0.1
azure-ai-projects>=2.1.0
rich>=13.0.0

```

Guardar y salir.

---

# 10. Instalar dependencias

```bash
uv pip install -r requirements.txt
```

---

# 11. Descargar un modelo local en Ollama

Ejemplo usando Qwen:

```bash
ollama pull qwen3.5:4b
```

También se puede usar:



---

# 12. Crear el script del agente

Crear archivo:

```bash
nano local_model.py
```

Contenido:

```python

import asyncio
import logging
from datetime import date

from agent_framework import Agent, tool
from agent_framework.openai import OpenAIChatClient

from rich.console import Console
from rich.logging import RichHandler
from rich.markdown import Markdown

console = Console()
logger = logging.getLogger("stage0")


@tool
def get_enrollment_deadline_info() -> dict:
    """Return enrollment timeline details for health insurance plans."""

    logger.info("[tool] get_enrollment_deadline_info()")

    return {
        "enrollment_opens": "2026-11-11",
        "enrollment_closes": "2026-11-30",
    }


client = OpenAIChatClient(
    base_url="http://localhost:11434/v1/",
    api_key="no-key-needed",
    model="qwen3.5:4b",
)

agent = Agent(
    client=client,
    instructions=(
        f"You are an internal HR helper. Today's date is {date.today().isoformat()}. "
        "Use the available tools to answer questions about benefits enrollment timing. "
        "Always ground your answers in tool results."
    ),
    tools=[get_enrollment_deadline_info],
)


async def main():
    console.print("[bold green]Interactive HR Agent[/bold green]")
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

    asyncio.run(main())
```

---

# 13. Ejecutar el agente

```bash
python local_model.py
```

---

# 14. Probar el agente

Ejemplo:

```text
When does benefits enrollment open?
```

Salida esperada:

```text
Benefits enrollment opens on 2026-11-11.
```

---

# 15. Salir del chat

```text
/q
```

---

# Estructura final del proyecto

```text
msft-agents-local/
│
├── .msft/
├── requirements.txt
└── stage0_local_model.py
```

---

# Comandos útiles

## Ver modelos instalados

```bash
ollama list
```

## Ver procesos de Ollama

```bash
ps aux | grep ollama
```

## Descargar otro modelo

```bash
ollama pull mistral
```

## Desactivar entorno virtual

```bash
deactivate
```
