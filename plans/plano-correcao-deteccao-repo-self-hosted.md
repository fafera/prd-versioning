# Plano — Corrigir detecção de repo/plataforma na skill prd-versioning

Origem: handoff `/tmp/handoff-prd-versioning-skill-fixes.md`
Repo da skill: `/home/fafera/.agents/skills/prd-versioning` (remote `github.com:fafera/prd-versioning`)
Branch atual: `refactor/tmp-efemero-sem-persistencia`
Arquivo central: `bin/prd-common` (todas as 5 chamadas git vivem aqui)

## Problema

A skill deriva o projeto-alvo do `git remote` do **cwd**. Quando o cwd é a própria
pasta da skill (que é um repo git apontando para github.com), dois erros aparecem ao
operar sobre um projeto GitLab self-hosted:

1. **Plataforma errada** — `_detect_platform` casa `//github` no remote da skill e
   retorna `github` em vez de `gitlab`.
2. **jq quebra com erro críptico** — mesmo com `PRD_PLATFORM=gitlab`, `_remote_host` /
   `_gitlab_project_path` leem o remote github; `glab api` bate em host/projeto errado,
   devolve não-JSON, e `jq` falha com `Invalid numeric literal at line 9`.

Workaround atual (funciona, mas frágil): rodar o bin por caminho absoluto **de dentro
da raiz do projeto** + `PRD_PLATFORM=gitlab`. Como o cwd reseta entre chamadas Bash, dá
certo desde que nunca se faça `cd` para a pasta da skill.

## Objetivo

Tornar a skill robusta a partir de qualquer cwd: âncora explícita de repo, guarda
contra repo/plataforma errados, e mensagens acionáveis no lugar de erros crípticos.

## Mudanças (todas em `bin/prd-common`, exceto a doc)

### 1. `_repo_dir()` + rotear todas as chamadas git por `git -C` — RAIZ DOS DOIS ERROS
Nova função:
```bash
_repo_dir() {
    [[ -n "$PRD_REPO_DIR" ]] && { echo "$PRD_REPO_DIR"; return; }
    git rev-parse --show-toplevel 2>/dev/null || echo "."
}
```
Trocar as 5 chamadas git para usar `git -C "$(_repo_dir)" ...`:
- `_repo_slug` (~37): `rev-parse --show-toplevel`
- `_remote_host` (~76-77): `remote get-url origin` e `remote -v`
- `_detect_platform` (~94): `remote -v`
- `_gitlab_project_path` (~156): `remote get-url origin`

Efeito: com `PRD_REPO_DIR` setado, a skill nunca mais depende do cwd. Sem ele, normaliza
para o toplevel a partir de subdiretórios (comportamento atual preservado).

### 2. Guarda repo-certo-vs-qualquer-repo — rede de segurança contra modo silencioso
Nova função, chamada nos caminhos de API antes de tocar a rede:
```bash
_guard_remote_matches() {
    local platform="$1" host
    host=$(_remote_host)
    case "$platform" in
        gitlab) [[ "$host" == "github.com" ]] && {
            echo "❌ PRD_PLATFORM=gitlab mas o remote resolvido é github.com (host=$host). Rode da raiz do projeto GitLab ou defina PRD_REPO_DIR/PRD_GITLAB_PROJECT." >&2; exit 1; } ;;
        github) [[ "$host" == *gitlab* ]] && {
            echo "❌ PRD_PLATFORM=github mas o remote resolvido aponta para GitLab (host=$host). Rode da raiz do projeto correto ou defina PRD_REPO_DIR." >&2; exit 1; } ;;
    esac
}
```
Motivo: `--show-toplevel` sozinho pode achar o repo ERRADO (não nenhum repo); a guarda
aborta com mensagem clara em vez de chamar a API com host/projeto trocados.

### 3. Validar JSON antes do `jq` — mata o erro críptico
Em `_gitlab_issue_json` (~204) e `issue_get_comments` (gitlab, ~288): capturar a saída,
rodar `jq empty`; se falhar, erro acionável citando host/projeto.
```bash
_gitlab_issue_json() {
    local iid="$1" out
    out=$(glab api $(_glab_api_host_flag) "projects/$(_gitlab_project_ref)/issues/$iid" 2>/dev/null)
    if ! printf '%s' "$out" | jq empty 2>/dev/null; then
        echo "❌ Resposta não-JSON da API GitLab (host=$(_remote_host), projeto=$(_gitlab_project_path)). Verifique auth (glab auth status), host e projeto." >&2
        return 1
    fi
    printf '%s' "$out"
}
```
Mesmo padrão para a chamada de notes em `issue_get_comments`.

### 4. Fallback de plataforma falha alto
No bloco de fallback de `_detect_platform` (~121): antes do chute por CLI disponível, se
`PRD_GITLAB_PROJECT` ou `GITLAB_HOST` estiverem setados, inclinar para `gitlab`. Manter
`unknown` → erro claro quando nada indica a plataforma.

### 5. Documentar no `SKILL.md`
Na seção "Platform Detection" / "Self-hosted GitLab": documentar
- rodar da raiz do projeto ou definir `PRD_REPO_DIR=/caminho/do/projeto`;
- `PRD_PLATFORM=gitlab` se a autodetecção falhar;
- exemplo self-hosted (host real `git.animaeducacao.com.br`).

## Prioridade
- **Itens 1 + 3** sozinhos eliminam os dois erros reportados → núcleo obrigatório.
- **Item 2** é a rede de segurança contra falha silenciosa.
- **Itens 4 + 5** são robustez/documentação.

## Verificação (antes de afirmar conclusão)
1. `bash -n bin/prd-common` — sintaxe.
2. Da raiz do projeto OneLearning: `PRD_PLATFORM=gitlab /home/fafera/.agents/skills/prd-versioning/bin/prd-sync 330` → deve imprimir o PRD v1.2 (caso de sucesso ainda funciona).
3. De DENTRO da pasta da skill com `PRD_PLATFORM=gitlab`: deve abortar com a mensagem da guarda (item 2), não com erro do jq.
4. De DENTRO da pasta da skill com `PRD_REPO_DIR=<raiz do projeto>`: deve funcionar igual ao passo 2.
5. Caso GitHub não-regredido: detecção continua retornando `github` num repo github real.

## Git
- Trabalhar no branch já existente `refactor/tmp-efemero-sem-persistencia` (ou novo branch se preferir).
- Commits em pt_BR, sem Co-Authored-By (CLAUDE.md).
- **Pedir permissão antes de qualquer push.**

## Pendência separada (fora desta skill, não bloqueia)
Alinhar `docs/substituicao-disciplinas/plano-unicidade-slug-disciplina-ulife.md` e a
memória `unicidade-slug-disciplina-ulife-design.md` à simplificação Post ID da issue
#330 (decisão do usuário ainda pendente).
