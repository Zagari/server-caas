

# Deploy das Aplicações do Updater de Clouflare e do Servidor Minecraft com Terraform e Ansible

Este guia descreve como provisionar uma infraestrutura completa na AWS com Terraform e, em seguida, usar o Ansible para implantar duas aplicações: um **servidor Minecraft Forge** e um serviço de **atualização de DNS para o Cloudflare**.

## Como Executar

Siga os passos abaixo para provisionar a infraestrutura e configurar os serviços.

### 1. Pré-requisitos no seu Computador

Antes de começar, certifique-se de que você tem as seguintes ferramentas instaladas na sua máquina local.

*   **Instalar Ansible:** Se ainda não tiver, instale-o com `pip`.
    ```bash
    python3 -m pip install ansible
    ```

*   **Instalar a coleção Docker do Ansible:** O playbook usa um módulo específico para Docker Compose que precisa ser instalado.
    ```bash
    ansible-galaxy collection install community.docker
    ```

### 2. Passo a Passo da Execução

A sequência abaixo deve ser seguida para uma implantação completa.

#### Passo 1: Configurar Segredos com Ansible Vault (Apenas na primeira vez)

Para gerenciar os tokens de API do Cloudflare de forma segura, usamos o Ansible Vault.

a. **Crie o arquivo de segredos criptografado**
No seu terminal, execute o comando abaixo. Ele pedirá que você crie e confirme uma senha, que será usada para proteger suas credenciais.

```bash
ansible-vault create secrets.yml
```

b. **Adicione suas variáveis secretas**
Dentro do editor de texto que abrir, cole o seguinte conteúdo no formato YAML, substituindo pelos seus valores reais:

```yaml
CLOUDFLARE_API_TOKEN: "your_token_here"
CLOUDFLARE_ZONE_ID: "your_zone_id_here"
CLOUDFLARE_RECORD_ID: "your_record_id_here"
CLOUDFLARE_RECORD_NAME: "subdomain.example.com"
MINECRAFT_RECORD_ID: "a_second_record_id_here"
```

Salve e feche o editor. Para editar este arquivo no futuro, use o comando 
```bash
ansible-vault edit secrets.yml`.
```

c. **Crie o arquivo de template `.env.j2`**
Este arquivo serve como um modelo para o arquivo `.env` que será criado no servidor. Crie o arquivo `.env.j2` com o seguinte conteúdo:

```jinja2
# Este arquivo é gerado automaticamente pelo Ansible. Não edite manualmente.
CLOUDFLARE_API_TOKEN="{{ CLOUDFLARE_API_TOKEN }}"
CLOUDFLARE_ZONE_ID="{{ CLOUDFLARE_ZONE_ID }}"
CLOUDFLARE_RECORD_ID="{{ CLOUDFLARE_RECORD_ID }}"
CLOUDFLARE_RECORD_NAME="{{ CLOUDFLARE_RECORD_NAME }}"
MINECRAFT_RECORD_ID="{{ MINECRAFT_RECORD_ID }}"
```

---

#### Passo 2: Provisionar a Infraestrutura na AWS

Com os arquivos de configuração prontos, crie os recursos na AWS.

a. **Aplique a configuração do Terraform:**
```bash
terraform apply -auto-approve
```

b. **Obtenha o IP público da instância:**
```bash
terraform output public_ip
```

c. **Atualize o arquivo de inventário do Ansible:**
Abra o arquivo `inventory.ini` e cole o endereço IP retornado no passo anterior no lugar de `seu_ip_publico_aqui`.

---

#### Passo 3: Configurar os Serviços com Ansible

Agora, execute os playbooks para configurar o servidor.

a. **Execute o Playbook do Cloudflare Updater:**
Este playbook irá clonar o repositório e criar o arquivo `.env` a partir do seu template e dos seus segredos.

```bash
ansible-playbook -i inventory.ini deploy_updt_cloudflare.yml --ask-vault-pass
```
O Ansible pedirá a senha do Vault (`Vault password:`). Digite a senha que você criou no Passo 1.

b. **Execute o Playbook do Servidor Minecraft:**
Este comando irá clonar o repositório do Minecraft, baixar os arquivos necessários e iniciar o servidor.

```bash
ansible-playbook -i inventory.ini deploy_minecraft.yml
```

Pronto! Ao final desses passos, sua instância EC2 estará no ar com ambos os serviços rodando em containers Docker.

---

### 3. Passo a Passo para Backup do Estado do Minecraft

O playbook se conecta à instância, compacta a pasta world/ e usa a ferramenta de linha de comando rclone para fazer o upload para o Dropbox. 

Passos para Implementação:

#### Passo 1: Instale e Configure o rclone no seu computador local para gerar um token de acesso do Dropbox de forma interativa.

a. Instale o rclone (ex: sudo apt install rclone ou via download).

b. Execute rclone config. Siga os passos para criar um novo "remote" do tipo Dropbox. Isso abrirá seu navegador para autorizar o acesso e, no final, salvará um token no arquivo ~/.config/rclone/rclone.conf. O "client_id" e o "client_secret" podem ser obtidos em https://www.dropbox.com/developers/apps

	•	Crie um app com tipo Scoped Access.
	•	Pegue o App key (Client ID) e App secret.

Você precisará do conteúdo deste arquivo de configuração para o Ansible.

#### Passo 2: Guarde o conteúdo de rclone.conf no Ansible Vault, para que seu token não fique exposto.

a. Abra seu arquivo secrets.yml com 
```bash
ansible-vault edit secrets.yml
```

b. Adicione uma nova variável com o conteúdo do arquivo de configuração:

```bash
# ... outras variáveis secretas ...
RCLONE_CONF_CONTENT: |
  [dropbox-remote-minecraft] # Nome do 
  type = dropbox
  token = {"access_token":"SEU_ACCESS_TOKEN_GERADO","token_type":"bearer","refresh_token":"SEU_REFRESH_TOKEN","expiry":"2023-10-27T15:00:00.123456Z"}
```

#### Passo 3: Novo Fluxo de Trabalho:

a. Para destruir: 
```bash
ansible-playbook -i inventory.ini backup_minecraft.yml --ask-vault-pass
````
b. Depois: 
```bash
terraform destroy -auto-approve
```
---

## Explicação Detalhada dos Playbooks

### Playbook: `deploy_updt_cloudflare.yml`
Este playbook é responsável por implantar o serviço que atualiza os registros DNS no Cloudflare.

*   **`vars_files: - secrets.yml`**: Instrui o Ansible a carregar as variáveis do arquivo `secrets.yml` (que está criptografado).
*   **`ansible.builtin.template`**: Usa o arquivo `src: .env.j2` como um template e o `dest: .../.env` como destino no servidor. Ele substitui as variáveis `{{ ... }}` pelos valores carregados do Vault.
*   **`mode: '0600'`**: Define permissões restritas para o arquivo `.env` gerado, garantindo que apenas o proprietário do arquivo (o usuário `ubuntu`) possa lê-lo e escrevê-lo.

### Playbook: `deploy_minecraft.yml`
Este playbook configura e inicia o servidor de Minecraft.

*   **`hosts: minecraft_server`**: Indica que o playbook deve ser executado nos hosts definidos no grupo `[minecraft_server]` do seu inventário.
*   **`become: yes`**: Permite que o Ansible execute tarefas com privilégios de superusuário (`sudo`), necessário para instalar pacotes e gerenciar serviços.
*   **`ansible.builtin.apt`**: Garante que pacotes essenciais como `git` e `docker` estejam instalados.
*   **`ansible.builtin.get_url`**: Baixa os arquivos de mods e mundo do Dropbox. A URL com `dl=1` força o download direto.
*   **`ansible.builtin.unarchive`**: Descompacta os arquivos baixados diretamente no servidor.
*   **`community.docker.docker_compose_v2`**: Módulo idempotente que gerencia o Docker Compose, garantindo que os containers estejam construídos e em execução (`state: present`).