# create_rbac_perfis.yml

Playbook para criar **organizações**, **perfis RBAC** (role definitions) e **times**
no Ansible Automation Platform 2.7 via Platform Gateway.

---

## Pré-requisitos

1. Instalar a collection:

```bash
cd playbooks/
ansible-galaxy collection install -r requirements.yml
```

2. Ter um token OAuth2 do Gateway com permissão para gerenciar organizações, roles e times.

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

### Perfis (role definitions)

| Perfil | Permissões |
|---|---|
| **Administrador** | Acesso total aos recursos da plataforma |
| **Suporte** | Leitura e execução (suporte funcional) |
| **Operador** | Criação, leitura, atualização e execução (contas de serviço; sem delete) |
| **Auditor** | Somente leitura (auditoria; perfil disponível, sem times nas orgs acima) |
| **Desenvolvedor** | Credenciais/inventário: leitura e execução · Projetos: leitura, execução e atualização · Templates: criação, leitura, atualização e execução |

Ajuste `vars/rbac.yml` antes de executar se precisar de outras orgs, permissões ou times.

---

## Execução

```bash
cd playbooks/
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
2. Cria/atualiza as organizações
3. Cria/atualiza as role definitions definidas em `aap_role_definitions`
4. Cria os times `{domínio}-{papel}` em cada organização
5. Atribui o perfil RBAC correspondente a cada time, no escopo da organização
6. Exibe um resumo ao final

---

## Depois de executar

Adicione usuários aos times `{domínio}-{papel}` de cada organização
(ex.: `BD-Administrador`, `DNS-Suporte`) para conceder o perfil correspondente.

Acesso platform-wide (superusuário) usa o papel gerenciado **Platform Admin**,
atribuído diretamente a usuários — não a times.
