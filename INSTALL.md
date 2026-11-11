# Instalação no Linux Mint (GPU NVIDIA RTX 50xx, 8 GB)

Passo a passo para um PC com Linux Mint 22.x (base Ubuntu 24.04) e uma RTX 5060 ou similar.
Confira a saída de cada passo antes de seguir. Tempo total: ~1 h.

## Requisitos

- Linux Mint 22.x, 64 bits (x86_64).
- GPU NVIDIA RTX 50xx (Blackwell) com 8 GB. Para RTX 40xx/30xx, ajuste `CUDA_ARCH` no passo 5.
- 16 GB de RAM ou mais, ~40 GB livres em disco, internet.
- Um usuário com `sudo`.

## 1. Driver NVIDIA

A RTX 50xx precisa do driver **575 ou mais novo, versão "open"**. Abra o **Gerenciador de
Drivers** do Mint, escolha o `nvidia-driver-5xx-open` mais recente da lista (575, 580 ou
mais novo), aplique e reinicie.

Se o Secure Boot estiver ligado, o instalador pede uma senha e, no reboot, aparece uma tela
azul (MOK): escolha *Enroll MOK*, digite a mesma senha e continue. Sem isso o driver não
carrega.

Depois do reboot:

```bash
nvidia-smi
```

Deve mostrar a GPU, a versão do driver e, no canto superior direito, `CUDA Version: 12.9`
(driver 575) ou `13.x` (driver 580+). Anote essa versão: ela vai para o passo 5.

## 2. Docker

O Mint usa nomes de versão próprios ("xia", "wilma"...), e o repositório do Docker precisa
do nome do **Ubuntu** em que ele se baseia. Por isso o comando abaixo usa `UBUNTU_CODENAME`:

```bash
for p in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove -y $p 2>/dev/null; done
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) stable" \
 | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

**Saia da sessão e entre de novo** (ou reinicie) para o grupo `docker` valer. Teste:

```bash
docker run --rm hello-world
```

## 3. NVIDIA Container Toolkit

Permite que os contêineres usem a GPU:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
 | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
 | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
 | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
docker run --rm --gpus all nvidia/cuda:12.9.1-base-ubuntu24.04 nvidia-smi
```

O último comando deve mostrar a GPU de dentro de um contêiner.

## 4. Obter o código

```bash
mkdir -p ~/llm-server/modelos && cd ~/llm-server
git clone <URL-DO-REPOSITÓRIO> pc-rtx-llm-server    # ou: unzip pc-rtx-llm-server.zip
cd pc-rtx-llm-server
chmod +x scripts/*.sh
```

Daqui em diante, todos os comandos rodam dentro de `~/llm-server/pc-rtx-llm-server`.

## 5. Configurar

```bash
cp .env.example .env
scripts/gerar-segredos.sh                   # 5 linhas "gerado:"
sed -i "s|^MODELS_DIR=.*|MODELS_DIR=$HOME/llm-server/modelos|" .env
ip -4 -br addr                              # anote o IP da placa de rede (ex.: 192.168.0.20)
nano .env
```

No `nano`, ajuste:

- `SERVER_NAME`: o IP anotado (ou `localhost`, se for usar só nesta máquina).
- `CUDA_VERSION`: no máximo a versão do `nvidia-smi` do passo 1. Use `12.9.1` com driver
  575; com 580+, `12.9.1` ou `13.0.1`.
- `CUDA_ARCH`: `120` para RTX 50xx (`89` para RTX 40xx, `86` para RTX 30xx).
- `HF_TOKEN`: opcional (evita limites de download do Hugging Face).

Se a rede local usar `10.254.x.x`, mude `DOCKER_SUBNET_*` para uma faixa livre.

## 6. Certificado e verificação

```bash
scripts/gerar-certificado.sh
scripts/verificar-host.sh       # deve terminar em "Pronto para seguir"
```

O aviso "nenhum modelo registrado ainda" é esperado neste ponto.

## 7. Baixar os modelos

```bash
scripts/adicionar-modelo.sh --listar Qwen/Qwen3-8B-GGUF
scripts/adicionar-modelo.sh qwen3-8b Qwen/Qwen3-8B-GGUF Q4_K_M --sem-raciocinio --padrao

scripts/adicionar-modelo.sh --listar ggml-org/gemma-3-4b-it-GGUF
scripts/adicionar-modelo.sh gemma3-4b ggml-org/gemma-3-4b-it-GGUF Q4_K_M
```

Cada um baixa o arquivo (~5 GB e ~2,5 GB) e o registra em `llamacpp/modelos.ini` e
`litellm/config.yaml`. Se o `--listar` não mostrar `Q4_K_M`, escolha a quantização de
tamanho parecido que aparecer.

## 8. Compilar e subir o llama.cpp

```bash
docker compose build llamacpp              # 5-20 min, só na primeira vez
docker compose up -d llamacpp
docker compose ps llamacpp                 # espere "(healthy)"
scripts/modelos.sh                         # qwen3-8b deve aparecer como "loaded"
nvidia-smi                                 # ~7 GB de VRAM em uso
```

## 9. Gateway, interface e nginx

```bash
docker compose up -d litellm
sleep 20 && curl -s http://127.0.0.1:4000/health/liveliness; echo    # "I'm alive!"
scripts/chave-webui.sh
docker compose up -d
docker compose ps                          # 5 serviços; open-webui leva 1-3 min
```

Abra `https://<SERVER_NAME>` (ou `https://localhost`), aceite o aviso do certificado e
**crie logo a conta de administrador** (o primeiro cadastro vira admin).

## 10. Testar

```bash
K=$(scripts/nova-chave.sh teste 10 1d)
for M in qwen3-8b gemma3-4b; do
  echo "== $M"
  curl -s http://127.0.0.1:4000/v1/chat/completions -H "Authorization: Bearer $K" \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"$M\",\"max_tokens\":100,\"messages\":[{\"role\":\"user\",\"content\":\"Diga olá.\"}]}" \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"
done
scripts/modelos.sh                         # agora gemma3-4b está "loaded"
curl -sk -o /dev/null -w "sem chave: %{http_code}\n" https://localhost:8443/v1/models   # 401
scripts/revogar-chave.sh teste
```

A segunda resposta demora mais: é a troca de modelo.

## Problemas conhecidos

| Sintoma | Causa | Solução |
|---|---|---|
| `nvidia-smi`: "couldn't communicate with the NVIDIA driver" | Driver não carregou (Secure Boot / MOK) | Passo 1; reinstalar o driver e fazer o *Enroll MOK* |
| `apt` do Docker: "Release file ... not found" / "xia" | Repositório com o nome do Mint | Usar `UBUNTU_CODENAME` (passo 2) |
| `could not select device driver "" with capabilities: [[gpu]]` | NVIDIA Container Toolkit ausente | Passo 3 |
| llama.cpp: "CUDA driver version is insufficient" | `CUDA_VERSION` maior que a do driver | Ajustar `CUDA_VERSION` e `docker compose build llamacpp` |
| llama.cpp: "no kernel image is available" | `CUDA_ARCH` errado para a GPU | Ajustar `CUDA_ARCH` e recompilar |
| "cudaMalloc failed: out of memory" / modelo `FALHOU` | Não coube na VRAM | Fechar programas que usam GPU; reduzir `c` em `modelos.ini` (ex.: 8192) ou usar quantização menor |
| Compilação falha em `ui-assets.cmake` / `loading.html` | Interface web embutida do llama.cpp | Já desligada no Dockerfile (`-DLLAMA_BUILD_UI=OFF`) |
| Depois de suspender o PC, o modelo para de responder | A GPU perde o estado CUDA na suspensão | `docker compose restart llamacpp` |
| Open WebUI: "Nenhum modelo disponível" depois de trocar a chave | O Open WebUI ignora o `.env` após a 1ª inicialização | `scripts/chave-webui.sh` e colar a chave em Painel do Administrador > Configurações > Conexões |
| Modelo novo não aparece no Open WebUI | LiteLLM não recarregou | `docker compose restart litellm` e recarregar a página |
| `docker compose up` não aplica mudança do `.env` | Variável com o mesmo nome exportada no terminal | Abrir um terminal novo e repetir |
| Primeira resposta lenta | Modelo sendo carregado na VRAM | Normal após ligar o PC ou trocar de modelo |
