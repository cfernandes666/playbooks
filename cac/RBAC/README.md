# create_orgs_teams.yml

Playbook para criar **organizações** e **times** no Ansible Automation Platform 2.7
via Platform Gateway, usando a collection `ansible.platform` com autenticação por
**token OAuth2**.

A atribuição de roles/permissões aos times fica para uma etapa posterior.

---

## Pré-requisitos

```bash
cd cac/RBAC/
ansible-galaxy collection install -r requirements.yml
```

Token OAuth2 do Gateway com permissão para gerenciar organizações e times.

---

## Autenticação

### Opção 1 — arquivo de variáveis

```bash
cp vars/aap_auth.yml.example vars/aap_auth.yml
```

```yaml
aap_hostname: "https://automation-gateway-aap.apps.exemplo.local"
aap_token: "<TOKEN_OAUTH2_DO_GATEWAY>"
aap_validate_certs: true
```

> Não versionar `vars/aap_auth.yml` com o token real (já está no `.gitignore`).

### Opção 2 — variáveis de ambiente

```bash
export AAP_HOSTNAME='https://automation-gateway-aap.apps.exemplo.local'
export AAP_TOKEN='<TOKEN_OAUTH2_DO_GATEWAY>'
export AAP_VALIDATE_CERTS='true'   # opcional
```

---

## Configuração (`vars/rbac.yml`)

| Organização | Times |
|---|---|
| **CESTI** | BD, Esteira, Multiplataforma, Mainframe, Armazenamento |
| **PRODUÇÃO** | BD, Multiplataforma, Mainframe, Observabilidade, Atendimento |
| **REDES** | DNS, ACL, Firewall, Balancer |

---

## Execução

```bash
cd cac/RBAC/
ansible-playbook create_orgs_teams.yml
```

Ou só com variáveis de ambiente:

```bash
export AAP_HOSTNAME='https://...'
export AAP_TOKEN='...'
ansible-playbook create_orgs_teams.yml
```

O playbook é idempotente (`state: present`): pode ser reexecutado após alterar `vars/rbac.yml`.
