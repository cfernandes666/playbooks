# create_rbac_perfis.yml

Playbook para criar **organizações**, **times** (Gateway) e **perfis RBAC**
customizados (Controller) no Ansible Automation Platform 2.7.

No AAP 2.7 as permissões `awx.*` (credential, inventory, project, templates)
existem na API do **Controller**, não no Gateway. Por isso o playbook:

1. Cria organizações e times via `ansible.platform` (Gateway)
2. Cria/atualiza role definitions customizadas em `/api/controller/v2/role_definitions/`
3. Atribui cada perfil ao time correspondente em `/api/controller/v2/role_team_assignments/`

---

## Pré-requisitos

1. Instalar a collection:

```bash
cd cac/RBAC/
ansible-galaxy collection install -r requirements.yml
```

2. Ter um token OAuth2 do Gateway com permissão para gerenciar organizações,
   times e roles do Controller.

---

## Autenticação

### Opção 1 — arquivo de variáveis

```bash
cp vars/aap_auth.yml.example vars/aap_auth.yml
```

Edite `vars/aap_auth.yml` e preencha:

```yaml
aap_hostname: "https://automation-gateway-aap.apps.exemplo.local"
aap_token: "<TOKEN_OAUTH2_DO_GATEWAY>"
aap_validate_certs: true
```

> Não versionar `vars/aap_auth.yml` com o token real.

### Opção 2 — variáveis de ambiente

```bash
export AAP_HOSTNAME='https://automation-gateway-aap.apps.exemplo.local'
export AAP_TOKEN='<TOKEN_OAUTH2_DO_GATEWAY>'
export AAP_VALIDATE_CERTS='true'   # opcional
```

---

## Configuração do RBAC

As organizações, perfis e times ficam em `vars/rbac.yml`.

Recursos no escopo dos perfis: **CREDENTIAL**, **INVENTORY**, **PROJECT**, **TEMPLATES**.

### Organizações e times

Cada organização define seus **domínios** (`teams`) e **papéis** (`roles`).
O playbook cria um time `{domínio}-{papel}` por combinação (ex.: `BD-Administrador`).

| Organização | Domínios | Papéis |
|---|---|---|
| **CESTI** | BD, Esteira, Multiplataforma, Mainframe, Armazenamento | Administrador, Suporte, Desenvolvedor, Operador |
| **PRODUÇÃO** | BD, Multiplataforma, Mainframe, Observabilidade, Atendimento | Suporte, Operador, Desenvolvedor |
| **REDES** | DNS, ACL, Firewall, Balancer | Suporte, Operador, Desenvolvedor |

### Perfis (role definitions no Controller)

| Perfil | Permissões |
|---|---|
| **Administrador** | Acesso total aos recursos da organização |
| **Suporte** | Leitura e execução (suporte funcional) |
| **Operador** | Criação, leitura, atualização e execução (contas de serviço; sem delete) |
| **Auditor** | Somente leitura (auditoria; perfil disponível, sem times nas orgs acima) |
| **Desenvolvedor** | Credenciais/inventário: leitura e uso · Projetos: leitura, uso e atualização · Templates: leitura, atualização e execução (criar JT exige project+inventory admin; WF tem `add`) |

Ajuste `vars/rbac.yml` antes de executar se precisar de outras orgs, permissões ou times.

---

## Execução

```bash
cd cac/RBAC/
ansible-playbook create_rbac_perfis.yml
```

Ou só com variáveis de ambiente:

```bash
export AAP_HOSTNAME='https://...'
export AAP_TOKEN='...'
ansible-playbook create_rbac_perfis.yml
```

O playbook é idempotente (`state: present`): pode ser reexecutado após alterar `vars/rbac.yml`.

---

## O que o playbook faz

1. Expande a matriz organização × domínio × papel a partir de `aap_organizations`
2. Cria/atualiza as organizações no Gateway
3. Cria/atualiza os times `{domínio}-{papel}` no Gateway
4. Cria/atualiza as role definitions customizadas no Controller
5. Aguarda sync Gateway→Controller e resolve IDs de orgs/times no Controller
6. Atribui o perfil RBAC correspondente a cada time, no escopo da organização
7. Exibe um resumo ao final

---

## Depois de executar

Adicione usuários aos times `{domínio}-{papel}` de cada organização
(ex.: `BD-Administrador`, `DNS-Suporte`) para conceder o perfil correspondente.

Acesso platform-wide (superusuário) usa o papel gerenciado **Platform Admin**,
atribuído diretamente a usuários — não a times.

### Conferir no Controller

```bash
curl -sk -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_HOSTNAME/api/controller/v2/role_definitions/?search=Administrador" \
  | jq '.results[] | {name, content_type, permissions}'
```
