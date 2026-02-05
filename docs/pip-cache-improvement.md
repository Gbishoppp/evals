# Melhoria: Cache do Pip no CI/CD e Dockerfiles

## Resumo

Esta melhoria implementa cache de pacotes pip em dois níveis:
1. **GitHub Actions (CI/CD)** - Principal foco da melhoria
2. **Dockerfiles** - Complemento para builds locais

O cache acelera significativamente o tempo de execução dos workflows e builds ao reutilizar pacotes já baixados.

## Problema

Sem cache, toda vez que um workflow do GitHub Actions é executado ou uma imagem Docker é construída, o pip precisa baixar novamente todos os pacotes da internet. Isso resulta em:

- **CI/CD lento**: Cada push/PR demora mais para validar
- **Custos elevados**: GitHub Actions cobra por minutos de execução
- **Uso desnecessário de banda**: Mesmos pacotes baixados múltiplas vezes
- **Builds inconsistentes**: Dependência de conexão com internet estável

---

## Parte 1: Cache nas GitHub Actions (Principal)

### Tecnologia Utilizada

A action `actions/setup-python` possui suporte nativo para cache do pip através do parâmetro `cache: 'pip'`. Quando habilitado:

1. O GitHub Actions salva o diretório de cache do pip (`~/.cache/pip`) após a execução
2. Em execuções futuras, restaura o cache antes de instalar dependências
3. O cache é invalidado automaticamente quando arquivos de dependências mudam

### Arquivos Modificados

#### 1. `.github/workflows/run_tests.yaml`

**Antes:**
```yaml
- name: Set up Python
  uses: actions/setup-python@v2
  with:
    python-version: 3.9

- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install pyyaml
    pip install pytest
    pip install -e .[torch]
```

**Depois:**
```yaml
- name: Set up Python
  uses: actions/setup-python@v4
  with:
    python-version: 3.9
    cache: 'pip'  # <-- Habilita cache do pip

- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install pyyaml
    pip install pytest
    pip install -e .[torch]
```

#### 2. `.github/workflows/test_eval.yaml`

**Antes:**
```yaml
- name: Set up Python
  uses: actions/setup-python@v2
  with:
    python-version: 3.9
```

**Depois:**
```yaml
- name: Set up Python
  uses: actions/setup-python@v4
  with:
    python-version: 3.9
    cache: 'pip'  # <-- Habilita cache do pip
```

### Como Funciona nas GitHub Actions

```
┌─────────────────────────────────────────────────────────────────┐
│                    PRIMEIRA EXECUÇÃO DO WORKFLOW                │
├─────────────────────────────────────────────────────────────────┤
│  1. Workflow inicia                                             │
│  2. setup-python verifica cache → NÃO ENCONTRADO                │
│  3. pip install baixa todos os pacotes da internet              │
│  4. Workflow executa os testes                                  │
│  5. setup-python SALVA ~/.cache/pip no cache do GitHub          │
│  6. Workflow finaliza                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   EXECUÇÕES SEGUINTES DO WORKFLOW               │
├─────────────────────────────────────────────────────────────────┤
│  1. Workflow inicia                                             │
│  2. setup-python verifica cache → ENCONTRADO ✓                  │
│  3. Cache é restaurado para ~/.cache/pip                        │
│  4. pip install usa pacotes do cache (download mínimo)          │
│  5. Workflow executa os testes MAIS RÁPIDO                      │
│  6. Workflow finaliza                                           │
└─────────────────────────────────────────────────────────────────┘
```

### Benefícios no CI/CD

| Métrica | Sem Cache | Com Cache | Melhoria |
|---------|-----------|-----------|----------|
| Tempo de instalação | ~2-3 min | ~20-30 seg | **80-90%** |
| Minutos faturados | Alto | Baixo | **Economia** |
| Confiabilidade | Depende do PyPI | Cache local | **Maior** |

### Invalidação do Cache

O cache é automaticamente invalidado quando:
- Arquivos `requirements.txt`, `setup.py`, `pyproject.toml` são modificados
- A versão do Python muda
- O cache expira (7 dias de inatividade)

---

## Parte 2: Cache nos Dockerfiles (Complementar)

### Tecnologia Utilizada: Docker BuildKit Cache Mounts

O Docker BuildKit permite montar diretórios de cache persistentes durante o build.

### Sintaxe

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=cache,target=/root/.cache/pip pip install pacote
```

### Arquivos Modificados

#### 1. `docker/flask-playwright/Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/playwright/python:v1.32.1-jammy

# Install Flask with pip cache
RUN --mount=type=cache,target=/root/.cache/pip pip3 install flask
```

#### 2. `docker/homepage/Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.8-slim-buster

# ... outras instruções ...

RUN --mount=type=cache,target=/root/.cache/pip pip3 install -r requirements.txt
```

#### 3. `docker/dc-evals-bash/Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1
FROM ubuntu:22.04

# ... outras instruções ...

# Enable pip cache for future pip installs
ENV PIP_CACHE_DIR=/root/.cache/pip
```

---

## Resumo das Alterações

| Arquivo | Tipo | Alteração |
|---------|------|-----------|
| `.github/workflows/run_tests.yaml` | CI/CD | Adicionado `cache: 'pip'` e atualizado para v4 |
| `.github/workflows/test_eval.yaml` | CI/CD | Adicionado `cache: 'pip'` e atualizado para v4 |
| `docker/flask-playwright/Dockerfile` | Docker | Adicionado BuildKit cache mount |
| `docker/homepage/Dockerfile` | Docker | Adicionado BuildKit cache mount |
| `docker/dc-evals-bash/Dockerfile` | Docker | Adicionado syntax e ENV para cache |

## Impacto Esperado

### GitHub Actions (Principal)
- **Redução de 80-90%** no tempo de instalação de dependências
- **Economia de custos** em minutos de execução
- **Maior confiabilidade** dos workflows

### Docker (Complementar)
- **Redução de 50-90%** no tempo de build local
- **Menor uso de banda** em desenvolvimento

## Referências

- [GitHub Actions - Caching dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [actions/setup-python - Caching packages](https://github.com/actions/setup-python#caching-packages-dependencies)
- [Docker BuildKit Cache Mounts](https://docs.docker.com/build/cache/)

---

*Documento criado para apresentação de melhoria no projeto OpenAI Evals*
