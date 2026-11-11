# Servidor LLM local para PC com GPU NVIDIA RTX (8 GB)

Serviço de LLMs abertos num PC com Linux Mint e GPU RTX 50xx de 8 GB, com chat no navegador
e API compatível com a OpenAI, acessível na própria máquina e pela rede local.

```
navegador ─── https://<PC>         ─> nginx ─> Open WebUI ─┐
cliente API ─ https://<PC>:8443/v1 ─> nginx ───────────────┼─> LiteLLM ─> llama.cpp (roteador)
na máquina ── http://127.0.0.1:4000/v1 ────────────────────┘              └─ 1 modelo na VRAM
```

- **llama.cpp (modo roteador):** carrega o modelo pedido e descarrega o anterior. Só um
  cabe nos 8 GB por vez, e a troca é automática.
- **LiteLLM:** gateway da API, com uma chave por pessoa, limites e registro de uso.
- **Open WebUI:** chat no navegador, com login.
- **nginx:** único serviço exposto à rede, com TLS.

Instalação: veja [INSTALL.md](INSTALL.md). Nos exemplos, `<PC>` é o IP ou nome da máquina
na rede (`SERVER_NAME` no `.env`).

## Modelos

| Nome na API | Modelo | Arquivo | Para quê |
|---|---|---|---|
| `qwen3-8b` | Qwen3 8B (Q4_K_M) | ~5,0 GB | Uso geral, melhor qualidade; raciocínio opcional |
| `gemma3-4b` | Gemma 3 4B (Q4_K_M) | ~2,5 GB | Respostas rápidas, contextos mais longos |

O `qwen3-8b` é carregado ao ligar o serviço. Pedir o outro (na API ou escolhendo no
Open WebUI) faz a troca sozinho: a primeira resposta depois da troca demora de alguns
segundos a ~1 minuto, enquanto o modelo sobe para a VRAM.

```bash
scripts/modelos.sh                       # lista os modelos e qual está carregado
scripts/modelos.sh carregar gemma3-4b    # troca manualmente
scripts/modelos.sh descarregar           # libera a VRAM (ex.: para jogar)
```

Evite alternar entre os dois modelos a cada mensagem: cada troca recarrega o modelo.

### Adicionar outros modelos

Qualquer GGUF do Hugging Face que caiba na VRAM:

```bash
scripts/adicionar-modelo.sh --listar Qwen/Qwen3-4B-GGUF              # vê arquivos e tamanhos
scripts/adicionar-modelo.sh qwen3-4b Qwen/Qwen3-4B-GGUF Q8_0 --sem-raciocinio
```

Regra prática para 8 GB: arquivos de até ~6 GB. Modelos de 7–9B em Q4_K_M, ou de 3–4B
em Q8_0. Opções do script: `--ctx N` (contexto), `--sem-raciocinio` (Qwen3 e similares),
`--padrao` (carrega ao ligar). O catálogo fica em `llamacpp/modelos.ini`.

## Uso pelo navegador

Abra `https://<PC>` (ou `https://localhost` na própria máquina). Com o certificado
autoassinado, aceite o aviso na primeira visita. Escolha o modelo no topo do chat.

## Uso pela API

Compatível com a API da OpenAI. Toda requisição precisa de uma chave
(`Authorization: Bearer sk-...`), criada com `scripts/nova-chave.sh`.

| De onde | Endereço | Certificado |
|---|---|---|
| Na própria máquina | `http://127.0.0.1:4000/v1` | não precisa |
| Pela rede local | `https://<PC>:8443/v1` | `nginx/certs/fullchain.pem` |

O `fullchain.pem` é público. A `privkey.pem` nunca sai da máquina.

### curl

```bash
curl -s http://127.0.0.1:4000/v1/chat/completions \
  -H "Authorization: Bearer $LLM_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"qwen3-8b","max_tokens":200,
       "messages":[{"role":"user","content":"Diga olá em uma frase."}]}'
```

Pela rede, troque o endereço por `https://<PC>:8443/v1/...` e acrescente
`--cacert fullchain.pem`.

### Python

```bash
pip install openai httpx
export LLM_API_KEY=sk-...
# só para uso pela rede:
export LLM_BASE_URL=https://<PC>:8443/v1
export LLM_CA_CERT=/caminho/fullchain.pem
```

```python
import os, httpx
from openai import OpenAI

cliente = OpenAI(
    base_url=os.getenv("LLM_BASE_URL", "http://127.0.0.1:4000/v1"),
    api_key=os.environ["LLM_API_KEY"],
    http_client=httpx.Client(verify=os.getenv("LLM_CA_CERT") or True, timeout=600),
)

r = cliente.chat.completions.create(
    model="qwen3-8b",                    # ou "gemma3-4b"
    messages=[{"role": "user", "content": "O que é aprendizado de máquina?"}],
    max_tokens=400,
)
print(r.choices[0].message.content)
```

Mais em [`exemplos/`](exemplos/):

| Arquivo | O que faz |
|---|---|
| `teste_conexao.py` | Confere endereço, certificado, chave e modelo (`LLM_MODEL=gemma3-4b` para o outro) |
| `uso_basico.py` | Pergunta simples, streaming, raciocínio, histórico, saída em JSON |
| `batch_cliente.py` | Lote de prompts de um JSONL, com paralelismo, novas tentativas e retomada |

### Raciocínio (qwen3-8b)

Vem desligado por padrão, para respostas diretas. Para ligar numa requisição:

```python
r = cliente.chat.completions.create(
    model="qwen3-8b", max_tokens=3000,
    messages=[{"role": "user", "content": "Quanto é 17 x 23?"}],
    extra_body={"chat_template_kwargs": {"enable_thinking": True}},
)
print(r.choices[0].message.reasoning_content)   # raciocínio
print(r.choices[0].message.content)             # resposta
```

Reserve `max_tokens` de sobra (2000+). O `gemma3-4b` não tem modo de raciocínio.

### Lote

```bash
# entrada.jsonl: {"id": "1", "prompt": "..."} por linha
LLM_MODEL=qwen3-8b python exemplos/batch_cliente.py entrada.jsonl saida.jsonl
```

O servidor atende 2 conversas ao mesmo tempo (`parallel = 2` em `llamacpp/modelos.ini`);
o script usa 4 requisições em paralelo para manter a GPU ocupada. Rodar de novo refaz só o
que faltou ou falhou.

### Limites

- Contexto: 16K tokens por modelo, dividido entre 2 conversas simultâneas (8K cada).
  Para conversas mais longas, use `parallel = 1` em `modelos.ini` (16K para uma conversa).
- Pela rede: até 20 requisições por segundo por IP.

## Administração

```bash
scripts/nova-chave.sh fulano 30 90d     # 30 req/min, válida por 90 dias
scripts/nova-chave.sh meu-script 0 365d # sem limite por minuto
scripts/revogar-chave.sh fulano
scripts/chave-webui.sh                  # troca a chave usada pelo Open WebUI
docker compose ps                       # estado dos serviços
docker compose logs -f --tail 50 llamacpp
nvidia-smi                              # uso de VRAM
docker compose stop                     # desliga tudo (libera a GPU)
docker compose start                    # liga de novo
```

Os serviços sobem sozinhos quando o PC liga. Para não ocupar a GPU o tempo todo, use
`scripts/modelos.sh descarregar` (libera a VRAM e mantém o serviço no ar; o próximo pedido
recarrega o modelo) ou `docker compose stop`.

## Segurança

- Só o nginx (80, 443, 8443) é exposto. Pela 8443 só a rota `/v1/` funciona; administração
  do LiteLLM apenas em `127.0.0.1:4000`.
- llama.cpp e Postgres ficam numa rede interna do Docker, sem acesso à internet.
- O Docker publica as portas por cima do firewall do Mint (ufw). Se o PC estiver numa rede
  não confiável, não suba o nginx (`docker compose stop nginx`) e use só `127.0.0.1:4000`.
- `.env` (segredos) e as chaves privadas ficam fora do git (`.gitignore`).
