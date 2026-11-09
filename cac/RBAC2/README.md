# create_rbac_perfis.yml

Playbook para criar **organizações** e **times** (Gateway) e atribuir
**roles gerenciadas** do Controller no Ansible Automation Platform 2.7.

Usa a collection `ansible.platform` com autenticação por **token OAuth2**.

## Por que roles gerenciadas (e não customizadas)?

Neste ambiente:

1. O **Gateway** só conhece permissões `shared.*` (org/team).
2. As permissões `awx.*` (credential, inventory, project, templates) existem no **Controller**.
3. Criar uma RoleDefinition customizada com `awx.*` no Controller falha com:

   ```text
   Failed to sync resource to resource server
   ```

   O Controller tenta fazer reverse-sync da role para o Gateway, e o Gateway
   rejeita porque não tem esses `api_slug`s.

Enquanto a sync de permissões remotas Gateway↔Controller não estiver completa,
o playbook atribui **roles gerenciadas org-scoped** já existentes no Controller.

---

## Pré-requisitos

```bash
cd cac/RBAC2/
ansible-galaxy collection install -r requirements.yml
```

Token OAuth2 do Gateway com permissão para gerenciar orgs, times e assignments.

---

## Autenticação

### Arquivo

```bash
cp vars/aap_auth.yml.example vars/aap_auth.yml
```

```yaml
aap_hostname: "https://automation-gateway-aap.apps.exemplo.local"
aap_token: "<TOKEN_OAUTH2_DO_GATEWAY>"
aap_validate_certs: true
```

### Ambiente

```bash
export AAP_HOSTNAME='https://...'
export AAP_TOKEN='...'
export AAP_VALIDATE_CERTS='true'
```

---

## Configuração (`vars/rbac.yml`)

### Organizações e times

O playbook cria um time `{domínio}-{papel}` por combinação (ex.: `BD-Administrador`).

| Organização | Domínios | Papéis |
|---|---|---|
| **CESTI** | BD, Esteira, Multiplataforma, Mainframe, Armazenamento | Administrador, Suporte, Desenvolvedor, Operador |
| **PRODUÇÃO** | BD, Multiplataforma, Mainframe, Observabilidade, Atendimento | Suporte, Operador, Desenvolvedor |
| **REDES** | DNS, ACL, Firewall, Balancer | Suporte, Operador, Desenvolvedor |

### Perfis → roles gerenciadas (Controller)

| Perfil | Roles gerenciadas atribuídas |
|---|---|
| **Administrador** | `Organization Admin` |
| **Suporte** | `Organization Execute` |
| **Operador** | Credential / Inventory / Project / JobTemplate / WorkflowJobTemplate **Admin** |
| **Auditor** | `Organization Audit` |
| **Desenvolvedor** | `Organization Execute` + Project / JobTemplate / WorkflowJobTemplate **Admin** |

Limitações em relação a um modelo 100% customizado:

- **Suporte** não ganha `use` de credential/inventory (só execute de templates).
- **Operador/Desenvolvedor** herdam **delete** das roles `* Admin`.
- Não existe `awx.add_jobtemplate`; criar JT exige admin de project+inventory.

---

## Execução

```bash
cd cac/RBAC2/
ansible-playbook create_rbac_perfis.yml
```

Idempotente: pode reexecutar após alterar `vars/rbac.yml`.

---

## O que o playbook faz

1. Expande organização × domínio × papel
2. Cria/atualiza organizações e times no Gateway (`ansible.platform.organization` / `ansible.platform.team`)
3. Resolve roles gerenciadas, orgs e times no Controller
4. Atribui cada managed role do perfil ao time, no escopo da organização
5. Exibe um resumo

---

## Depois de executar

Adicione usuários aos times `{domínio}-{papel}` (ex.: `BD-Administrador`).

Acesso platform-wide continua sendo **Platform Admin** (usuário, não time).

### Conferir assignments

```bash
curl -sk -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_HOSTNAME/api/controller/v2/role_team_assignments/?page_size=5" \
  | jq '.results[] | {role: .summary_fields.role_definition.name, team: .summary_fields.team.name}'
```
