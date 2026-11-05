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

Por padrão o playbook garante:

- **Organizações:** Seguranca, Automacao, Operacao
- **Perfis:** Admin, Operador, Desenvolvedor
- **Times:** um time por perfil em cada organização (ex.: `Seguranca/Admin`)

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

1. Cria/atualiza as organizações definidas em `aap_organizations`
2. Cria/atualiza as role definitions definidas em `aap_role_definitions`
3. Cria um time por perfil em cada organização (`aap_teams` × `aap_organizations`)
4. Atribui o perfil RBAC correspondente a cada time, no escopo da organização
5. Exibe um resumo ao final

---

## Depois de executar

Adicione usuários aos times **Admin**, **Operador** ou **Desenvolvedor** de cada
organização para conceder o perfil correspondente.

Acesso platform-wide (superusuário) usa o papel gerenciado **Platform Admin**,
atribuído diretamente a usuários — não a times.
