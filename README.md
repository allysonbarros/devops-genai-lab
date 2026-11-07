# 🚀 Projeto Integrador: DevOps com GenAI (versão GitHub Actions + GroqCloud)

Especialização em Engenharia de Software: Automação e Inovação com Inteligência Artificial Generativa (AKCIT/UFG), Unidade 5.

Adaptação do laboratório original em GitLab ([tornis1/devops-genai-lab](https://gitlab.com/tornis1/devops-genai-lab)) para **GitHub Actions**, usando o **GroqCloud** (camada gratuita) como motor de IA generativa.

## 🎯 Objetivo
Demonstrar a integração entre um pipeline de CI/CD e um agente de IA generativa que lê o log de uma falha, correlaciona com o diff do Pull Request e sugere a correção. A decisão de aplicar continua humana.

> A GenAI sugere. O humano valida. A automação executa.

## 🛠️ Arquitetura

1. **Aplicação:** `OrderService` (Python/Flask) com um bug intencional de tipagem (`TypeError` ao somar `float` com `str`).
2. **Pipeline:** `.github/workflows/ci.yml` com dois jobs.
   * `unit_test` roda o pytest, salva a saída em `test_output.log` e publica como artefato.
   * `ai_debugger` só roda quando `unit_test` falha em um Pull Request (`if: failure()`).
3. **Agente IA:** `genai_analyst.py` lê o log, busca o diff do PR pela API do GitHub, mascara segredos e consulta o LLM no GroqCloud.
4. **Feedback:** o diagnóstico é publicado como comentário no PR e no resumo da execução do workflow.

| Original (GitLab) | Esta versão (GitHub) |
|---|---|
| `.gitlab-ci.yml` com stages `test` e `genai_analysis` | Workflow com jobs `unit_test` e `ai_debugger` |
| `when: on_failure` | `needs: unit_test` + `if: failure()` |
| CI/CD Variables | Secrets and variables > Actions |
| `CI_JOB_TOKEN` | `GITHUB_TOKEN` com `pull-requests: write` |
| Trace buscado pela API | Log passado como artefato entre jobs |
| Merge Request | Pull Request |
| OpenAI (`gpt-4o`, pago) | GroqCloud (`llama-3.3-70b-versatile`, camada gratuita) |

## 📋 Pré-requisitos

1. Crie uma conta em [console.groq.com](https://console.groq.com) e gere uma chave em **API Keys** (começa com `gsk_`).
2. No repositório do GitHub, vá em **Settings > Secrets and variables > Actions > New repository secret**.
   * Name: `GROQ_API_KEY`
   * Secret: sua chave do GroqCloud
3. (Opcional) Na aba **Variables** do mesmo menu, crie `GROQ_MODEL` para trocar o modelo, por exemplo `openai/gpt-oss-120b`.

Nunca coloque a chave no código. O `GITHUB_TOKEN` é criado automaticamente a cada execução.

## 🏃 Como testar a automação

1. Crie uma branch a partir da `main` (ex.: `feature/nova-logica`).
2. Faça uma alteração trivial (ex.: um comentário no `app.py`) e faça o commit.
3. Abra um **Pull Request** para a `main`.
4. Na aba **Checks** do PR:
   * o job `unit_test` falha (❌);
   * o job `ai_debugger` é disparado automaticamente.
5. Volte à aba **Conversation** do PR e veja o comentário do bot.

### Rodar localmente

```bash
pip install -r requirements.txt
pytest test_app.py -v 2>&1 | tee test_output.log
export GROQ_API_KEY=gsk_...
python genai_analyst.py test_output.log   # sem PR, imprime a análise no console
```

## 🐞 Problemas encontrados no código original (GitLab)

Ao portar o laboratório, identifiquei pontos que impedem o fluxo original de funcionar como descrito:

1. **`allow_failure: true` anula o `when: on_failure`.** Para o GitLab, um job com falha permitida conta como sucesso, então o `ai_debugger` nunca é disparado.
2. **`when` no nível do job junto com `rules`.** O GitLab rejeita essa combinação na validação do YAML. O `when: on_failure` precisa ficar dentro da regra.
3. **`import gitlab` sem `python-gitlab` no `requirements.txt`.** O script quebra com `ModuleNotFoundError` antes de começar, e o módulo nem é usado.
4. **`CI_JOB_TOKEN` enviado no header `PRIVATE-TOKEN`.** O job token usa o header `JOB-TOKEN` e não tem permissão para ler traces de jobs nem para comentar em MRs. O caminho é um Project Access Token com escopo `api`.
5. **O diff não é coletado.** O material descreve a correlação entre log e diff, mas o script envia só o log.
6. **Comentário Python dentro do f-string do prompt.** O texto `# Enviando apenas os últimos 2000 caracteres...` vai para o LLM como parte do prompt.

Esta versão trata todos esses pontos e acrescenta mascaramento de segredos e uma instrução de defesa contra injeção de prompt vinda do log ou do diff.

## ⚠️ Isenção de Responsabilidade
Este projeto usa IA para sugerir correções. As sugestões devem sempre ser validadas por um humano antes de chegar à produção. No caso deste laboratório, a sugestão típica `float(tax_rate)` resolve o sintoma, mas a correção mais robusta é validar o tipo na entrada da API e considerar `Decimal` para valores monetários.
