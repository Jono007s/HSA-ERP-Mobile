# HSA-ERP-Mobile
Integração web em Python para enviar HSP via TCP/IP para impressoras HSA usando dados SQL do ERP.

# 🔌 Integração HSA ERP Mobile

Sistema **web** em **Python (puro)** para operação de **impressão industrial** com **controladores/impressoras HSA** (qualquer modelo com **TCP/IP**), consumindo dados de **ERP via SQL** (**MySQL** ou **SQL Server**).
Funciona em **servidor central**; os **clientes** (Android, Windows, Linux) usam via **navegador**. Homologado e em uso em **coletores Android Unitech EA520**.

> **Protocolo:** TCP/IP (envio raw) • **Formato do job:** HSP • **Template:** único (parametrizável por projeto).
> **Bipagem:** modo teclado (wedge) no coletor • **Simbologia:** Code128.

---

## 🧭 Visão geral (do “chão de fábrica”)

**Fluxo resumido:**

1. Operador **bipa a OP** no coletor (EA520) → o input do card recebe o código automaticamente.
2. Clica em **Buscar** (ou a tela dispara a busca conforme configuração).
3. O card exibe a **conferência** do item (ex.: *Material, Descrição, Qtd OP, Situação*).
4. Operador confirma clicando em **Enviar** → sistema monta o **HSP** e envia por **TCP/IP** ao IP/porta da impressora HSA.
5. O card muda para **Trabalhando** (bloqueia ações enquanto processa) e depois retorna ao estado **Livre**.
6. Se necessário, o operador pode **Parar** → a UI confirma e o card volta ao estado **Livre**.

---

## 🖥️ Layout e UX (Cards)

- **Grid responsivo**
  - **Desktop:** cards lado a lado (grid), sem rolagem horizontal.
  - **Coletor Android (EA520):** coluna única com **rolagem vertical**.
- **Cada card representa uma impressora** e exibe:
  - **Cabeçalho:** *Nome* (ex.: `494`, `413`, `232`…), **IP** (ex.: `192.168.1.155`) e **Porta**.
  - **Campo de entrada:** “**Digite a OP**” (recebe a bipagem do coletor).
  - **Botões:** **Buscar**, **Enviar**, **Parar** (mostrado quando aplicável), com travas por estado.
  - **Painel de conferência** (quando há dados): *Material, Descrição, Qtd OP, Situação*.
  - **Badge de status:** **Livre**, **Trabalhando** ou **Em Conferência**.

**Estados visuais do card**

| Estado         | O que indica                                              | Ações/Botões                                  |
|----------------|-----------------------------------------------------------|-----------------------------------------------|
| **Livre**      | Pronto para receber nova OP                               | `Buscar` ativo; `Enviar` exige conferência    |
| **Conferência**| Dados carregados (Material/Descrição/Qtd/Situação)        | `Enviar` ativo; `Parar` oculto                |
| **Trabalhando**| HSP enviado / enviando (ocupado)                          | `Parar` ativo; `Buscar`/`Enviar` desabilitados|
| **Erro**       | Falha (busca ou envio)                                    | Mensagem; volta a **Livre**                   |

> O sistema aplica **feedback imediato** (cores/labels) e **desabilita** botões durante operações críticas, evitando duplo clique.

---

## 🔎 Bipagem (EA520) e leitura de dados

- **Modo de entrada:** **teclado wedge** (o leitor injeta dígitos no input do card).
- **Simbologia:** **Code128**.
- **Disparo da busca:** por padrão, **após bipar o operador clica em “Buscar”** (há suporte a autotrigger por projeto).
- **Feedback do coletor:** o EA520 emite o **bip** local ao ler o código (som do hardware).

---

## 📋 Conferência e validações

Após **Buscar**, o card exibe o bloco de conferência (como na tela em produção):

- **Material**
- **Descrição** (quebra automática de linha)
- **Qtd OP**
- **Situação** (ex.: *Finalizada*, *Liberada*, etc.)

**Validações:**

- Por padrão, **não bloqueamos o Enviar** por situação — **as regras são configuráveis** por projeto (ex.: impedir envio se Situação ≠ *Liberada*).
- O **template de impressão é único**; os dados vêm de **qualquer ERP** que ofereça **SQL** (MySQL/SQL Server).

---

## 📤 Envio de impressão (HSP via TCP/IP)

- Montagem do **payload HSP** (template único + variáveis) e envio via **socket TCP** ao **IP/Porta** da HSA do card.
- **Sem driver do SO:** envio **raw** por TCP/IP — independe de spool/fila do sistema operacional.
- **Cópia do HSP:** **fica na própria impressora** (quando aplicável); o sistema não persiste localmente por padrão.

**Pós-envio**

- O card entra em **Trabalhando** e **bloqueia** novas ações.
- Ao concluir, **retorna automaticamente** a **Livre** (sem recarregar a página).

---

## 🛑 Parada

- Ação **Parar**:
  - A UI **confirma** a intenção.
  - O estado do card volta a **Livre** (libera input e botões).
  - Registra a ação em **log**.

---

## 🖨️ Cadastro de Impressoras

Cadastro **objetivo** para operação TCP/IP:

| Campo    | Descrição                                  |
|--------- |--------------------------------------------|
| **Nome** | Identificador visível do card (ex.: `494`) |
| **IP**   | Endereço do controlador HSA                |
| **Porta**| Porta TCP (ex.: `9100`)                    |

- **Sem vinculação** obrigatória por usuário/setor.
- **Sem impressora padrão** por sessão.
- Ao cadastrar, a impressora **vira um novo card** no grid.

---

## 🔄 Atualização de status (tempo real prático)

A UI mantém coerência por **dois mecanismos**:

1. **Eventos de ação** (Buscar/Enviar/Parar) atualizam **imediatamente** o card acionado.
2. **Polling leve** sincroniza **todos os cards** periodicamente (captura ações de outros operadores).

O intervalo de atualização é **ajustável** por projeto (equilíbrio entre responsividade e carga).

---

## 👤 Perfis e segurança

- **Perfis**
  - **Operador:** tela de **operação** (cards).
  - **Dashboard:** **somente leitura** (visualizar o que está sendo impresso).
- **Acesso:** **intranet** (rede interna).

---

## 🧾 Logs e rastreabilidade

O sistema registra:

- **Impressora** (Nome/IP/Porta)
- **Horário**
- **OP** e **Produto** (dados da conferência)
- **Ação** (Buscar, Enviar, Parar) e **resultado** (OK/erro)
- **Usuário/Perfil** (quando aplicável)
- **IP do cliente** (origem)

> O **HSP não é arquivado** pelo sistema — a cópia **fica na impressora**. Se necessário, é possível ativar **espelhamento opcional** por projeto.

---

## 🧱 Modularidade (alto nível)

Organização em camadas coesas (sem depender de nomes físicos de arquivos):

- **Dados (SQL):** conexão/consultas ao ERP em **MySQL** ou **SQL Server** (selecionável por ambiente).
- **Impressão (HSP/TCP):** montagem do **HSP** (template único) + envio **TCP/IP** para cada HSA.
- **Aplicação (Serviço):** orquestra **Buscar → Conferir → Enviar → Trabalhando/Livre**, gerencia estados e logs.
- **UI (Web):** grid de **cards**, inputs, botões e feedback visual; pronta para **EA520 (Android)** e **Desktop**.

Benefícios:

- **Adicionar impressoras** sem mexer na lógica central (cada uma vira um card).
- **Trocar fonte de dados** (MySQL ↔ SQL Server) por configuração.
- **Ajustar validações/layout** por projeto, sem quebrar o núcleo.

---

## 🌐 Arquitetura de implantação

```
[ Usuários (Android EA520 / Windows / Linux) ]
                 │  HTTP/HTTPS (intranet)
                 ▼
           [ Servidor Web ]
        (Python • UI + API)
                 │  TCP/IP
                 ▼
         [ Controladores HSA ]
       (qualquer modelo HSA com TCP)
```

- **Servidor central** hospeda a aplicação web.
- **Clientes** acessam via **navegador** (coletor EA520, PCs, etc.).
- **Envio HSP** é **direto do servidor** para **cada HSA** (IP/Porta por card).

---

## 📌 Compatibilidade

- **ERP:** qualquer que exponha dados via **SQL** (**MySQL** ou **SQL Server**).
- **HSA:** **qualquer modelo** que aceite **HSP via TCP/IP** (porta configurável).
- **Dispositivos:** Android (EA520 homologado), Windows, Linux — acesso via navegador.

---

## 🎨 UI — Cards (rótulos, cores e microcopy)

- **Grid responsivo**
  - **Desktop:** cards em colunas (sem rolagem horizontal).
  - **Coletor (Android/EA520):** coluna única com **rolagem vertical**.

- **Card (estrutura)**
  - **Título do card:** **Nome** da impressora (ex.: `TI`, `494`)
  - **Subtítulo:** `IP: 192.168.1.68` (exemplo)
  - **Campo de entrada:** input para **OP** (recebe bipagem)
  - **Botão primário 1:** **Buscar**
  - **Painel de conferência (quando houver dados):**
    - **Material:** código do item
    - **Descrição:** texto longo (quebra automática)
    - **Qtd OP:** quantidade total
    - **Situação:** status da OP (ex.: `Finalizada`, `Liberada`)
  - **Botão primário 2:** **Enviar**
  - **Badge de status:** **Livre** / **Trabalhando**

- **Cores & estados**
  - **Card:** gradiente **verde** (padrão atual); variações **âmbar/alaranjado** quando ocupado.
  - **Botões (Buscar/Enviar):** **azul**; ficam **desabilitados** durante operações.
  - **Painel de conferência:** “pill” **verde-claro**.
  - **Badge “Livre”:** **verde** suave; **“Trabalhando”**: **âmbar**.

- **Microcopy**
  - **Buscar:** “Buscar”
  - **Enviar:** “Enviar”
  - **Parar:** “Parar” + confirmação (“Confirmar parada?”)
  - **Estados:**
    - **Livre** → pronto para nova OP
    - **Conferência** → mostra Material/Descrição/Qtd/Situação (habilita **Enviar**)
    - **Trabalhando** → bloqueia inputs/botões até concluir (retorna a **Livre** automaticamente)
  - **Erros comuns:** “Falha ao buscar OP” / “Falha ao enviar para a impressora”

---

## 🧩 Dependências (explicação objetiva)

> Projeto **Python** rodando no **servidor**; clientes acessam via **navegador**. Abaixo os blocos e porquês.

- **Servidor Web**
  - **Flask** (ou equivalente já no projeto): expõe a **UI** e **endpoints internos** (buscar OP, montar HSP, enviar TCP).
  - **Jinja2** (com Flask): renderiza **templates HTML** da UI.

- **Banco de Dados (ERP via SQL)**
  - **MySQL** → **PyMySQL** (driver puro) **ou** **mysqlclient** (lib nativa).  
    *Prática comum:* **PyMySQL** pela portabilidade.
  - **SQL Server** → **pyodbc** (requer driver ODBC apropriado no SO).  
    *Observação:* Em Android/EA520, o servidor faz a consulta — o coletor usa o navegador.

- **Impressão HSA (TCP/IP + HSP)**
  - **socket** (stdlib): abre **conexão TCP** e envia o **payload HSP** *raw* ao **IP/porta** da HSA.
  - **Normalização/encoding** (stdlib): padroniza conteúdo HSP (ASCII/UTF-8 sem BOM) para o firmware.

- **Ambiente & Configuração**
  - **python-dotenv**: carrega **variáveis de configuração** (IPs/portas, credenciais SQL, timeouts) sem hardcode.
  - **logging** (stdlib) + **RotatingFileHandler**: **logs rotativos** (quem, quando, OP/impressora/resultado).

- **Qualidade de vida (opcional)**
  - **tabulate**: saídas **legíveis** em modo CLI/debug.
  - **colorama** (Windows): **cores** em logs/CLI.
  - **watchdog** (se aplicável): monitoramento de eventos locais (não requerido no fluxo TCP puro).

**Notas**
- **Sem driver de impressora**: envio **direto por TCP**; independe de spool do SO.
- **Template único (HSP)** simplifica manutenção; variações ficam nos **dados**.
- **Multibanco**: **MySQL** e **SQL Server** selecionáveis por configuração.
- **Logs completos**: impressora (Nome/IP/Porta), horário, OP/produto, ação (Buscar/Enviar/Parar), usuário/perfil e IP do cliente — o **HSP não é arquivado** pelo sistema (fica na impressora).

---

## 📜 Autor

**João Pedro Cardoso**  
*Consultor/Desenvolvedor – Automação & ERP*

- 📧 **joaopedro.vieiracardoso6@gmail.com**
- 🔗 **LinkedIn:** https://www.linkedin.com/in/joa0o/
