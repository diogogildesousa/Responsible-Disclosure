---

# Relatório de Segurança - Responsible Disclosure

### Página 1

**Relatório de Segurança [EMPRESA REDACTADA]**

**RELATÓRIO DE SEGURANÇA**

**Responsible Disclosure**

**[EMPRESA REDACTADA]**

**[domínio-redactado]**

| Campo | Informação |
| :--- | :--- |
| **Data** | 3 de março de 2026 |
| **Autor** | Diogo Sousa |
| **Contacto** | [REDACTADO]<br>[REDACTADO] |
| **Tipo** | *Responsible Disclosure* |

---


### Página 2

**Relatório de Segurança [EMPRESA REDACTADA]**

**Introdução**

Enquanto cliente de [EMPRESA REDACTADA], durante a utilização normal da aplicação web ([domínio-redactado]), identifiquei situações que considero relevantes partilhar com a equipa técnica. Este relatório apresenta as minhas observações de forma organizada, separando factos confirmados de observações que carecem de verificação adicional.

Todas as observações foram feitas através das ferramentas de desenvolvimento do browser (F12) durante utilização normal da plataforma com a minha conta pessoal. Não utilizei ferramentas ofensivas nem tentei explorar qualquer vulnerabilidade.

Para a análise técnica (*headers* HTTP, código fonte, e estrutura da API), contei com a assistência de ferramentas de inteligência artificial, que me ajudaram a interpretar os dados e a formular recomendações de correção.

**Resumo**

**Factos Confirmados**

| # | Severidade | Descrição | Categoria |
| --- | --- | --- | --- |
| F1 | CRÍTICA | Dados pessoais de todos os utilizadores visíveis no browser | Dados/RGPD |
| F2 | ALTA | Credenciais de admin do sistema de acesso expostas no *frontend* | Acesso Físico |
| F3 | ALTA | Chave e endpoint de pagamentos expostos no código fonte | Pagamentos |
| F4 | MEDIA | API de pagamentos *deprecated* desde setembro 2024 | Pagamentos |
| F5 | MEDIA | *Headers* de segurança HTTP não configurados | Configuração |

**Observações (a confirmar)**

| # | Tipo |  | Descrição | Categoria |
| --- | --- | --- | --- | --- |
| 01 | OBS |  | Persistência de sessão / *tokens* Firebase acessíveis no IndexedDB | Sessão |
| 02 | OBS |  | Possivel falha no fluxo de confirmação de pagamento com atraso | Pagamentos |
| 03 | OBS |  | Relógio do dispositivo de controlo de acesso dessincronizado | Infraestrutura |


---

### Página 3

**Relatório de Segurança [EMPRESA REDACTADA]**

**Factos Confirmados - Detalhes**

**F1 - Exposição de dados pessoais de todos os utilizadores**

* **O que observei:** Ao aceder à secção "Minhas Reservas" da aplicação com a minha conta, abri as ferramentas de desenvolvimento do browser (F12, separador Consola). Na consola aparecem mensagens do tipo "MapEntry(...)" que contêm dados de reservas de outros clientes não apenas as minhas.
* **Dados visíveis:** Endereços de email pessoais de outros clientes, PINs de 6 dígitos associados a cada reserva, datas e horas das reservas, valores pagos, e estado do pagamento.
* **Como reproduzir:** Fazer login em [domínio-redactado]. Abrir F12 e ir ao separador Consola. Clicar em "Minhas Reservas". Observar as mensagens na consola contêm campos como client_email, pin, price, state, dateMade, de diversos utilizadores.

*[Imagem redactada para proteção da entidade]*

*Screenshot da consola com dados pessoais de utilizadores - emails, PINs e dados de reservas redactados por conter dados pessoais de terceiros (RGPD)*

* **Porque é relevante:** A aplicação aparenta carregar todos os registos da base de dados para o browser do cliente, fazendo a filtragem apenas no *frontend*. Isto significa que os dados pessoais de todos os clientes são enviados para o browser de qualquer utilizador autenticado. Ao abrigo do RGPD, dados pessoais de terceiros não devem ser acessíveis desta forma.
* **Recomendação (sugerida por IA):** Implementar *Firestore Security Rules* que restrinjam o acesso de cada utilizador apenas aos seus próprios documentos. A query no *frontend* deve filtrar por *user* ID, e as *rules* do servidor devem reforçar essa restrição (ex: allow read: if request.auth.uid == resource.data.userId).

**F2 - Credenciais de admin do sistema de acesso expostas no frontend**

* **O que observei:** Quando clico no botão "Abrir Porta" na aplicação (disponível quando tenho uma reserva ativa), abre-se um novo separador no browser. O URL completo, visível na barra de endereço, contém as credenciais de administrador do dispositivo de controlo de acesso:
`http://admin:[PASSWORD_REDACTADA]@:8080/cgi-bin/accessControl.cgi?action=openDoor&channel=1&UserID=[REDACTADO]&Type-Remote`
* **Credenciais expostas:** Username: admin / Password: [REDACTADA]


---

### Página 4

**Relatório de Segurança [EMPRESA REDACTADA]**

*[Imagem redactada para proteção da entidade]*

*Screenshot do painel de rede com URL contendo credenciais - password e hostname redactados*

**Pontos adicionais:**

* O URL utiliza HTTP (não HTTPS) na porta 8080, sem encriptação
* Utiliza um hostname DDNS público, acessível a partir da internet
* O parâmetro UserID parece ser fixo
* Qualquer utilizador que inspecione o URL ao clicar em "Abrir Porta" tem acesso às credenciais de administrador do dispositivo
* Atualmente o endpoint retorna *timeout* - o dispositivo pode estar offline ou o DDNS expirado
* **Recomendação (sugerida por IA):** Não expor dispositivos IoT diretamente na internet. O controlo da porta deveria ser mediado por um servidor *backend* autenticado que valide a sessão do utilizador e a existência de uma reserva válida antes de enviar o comando ao dispositivo. As credenciais nunca devem estar presentes no *frontend*. O dispositivo deveria estar acessível apenas na rede local.

**F3 - Chave e endpoint de pagamentos expostos no código fonte**

* **O que observei:** Ao inspecionar o ficheiro principal da aplicação (main.dart.js) através do separador "Depurador" (F12), pesquisei pelo nome do serviço de pagamentos e encontrei a chave de integração, assim como o URL completo do endpoint, diretamente no código.
* **Chave e endpoint encontrados:**
`MbWayKey=... .asmx/SetPedidoJSON?MbWayKey=... &canal=...&referencia=...&valor=...&rtim=...&email=...&descricao=...`

*[Imagem redactada para proteção da entidade]*

*Screenshot do código fonte com chave de pagamentos e endpoint - valores redactados*

* **Porque é relevante:** A chave de integração com o serviço de pagamentos e a estrutura completa do endpoint estão acessíveis a qualquer pessoa que visite o site e inspecione o código fonte. Chaves de integração com serviços de pagamento devem ser mantidas exclusivamente no servidor.
* **Recomendação (sugerida por IA):** Mover toda a lógica de pagamento para um servidor *backend*. O *frontend* envia o pedido ao *backend*, que comunica com a API do serviço de pagamentos usando a chave armazenada em variáveis de ambiente no servidor.


---

### Página 5

**Relatório de Segurança [EMPRESA REDACTADA]**

**F4 - API de pagamentos deprecated**

* **O que observei:** O endpoint utilizado pela aplicação corresponde à versão antiga da API MBWay do serviço de pagamentos utilizado.
* **Confirmação:** O próprio fornecedor do serviço classifica esta API como "*Deprecated*" na sua documentação oficial, indicando que uma nova API está disponível desde setembro de 2024.
* **Porque é relevante:** APIs *deprecated* podem deixar de funcionar a qualquer momento e tipicamente não recebem atualizações de segurança.
* **Recomendação (sugerida por IA):** Migrar para a nova versão da API, idealmente com a lógica de pagamento no *backend* (resolvendo também F3 simultaneamente).

**F5 - Headers de segurança HTTP não configurados**

* **O que observei:** Com recurso a um script de análise automatizada, verifiquei que o servidor não envia vários *headers* de segurança HTTP que são considerados boas práticas. Em termos simples, estes *headers* são instruções que o servidor envia ao browser a dizer como se deve comportar para proteger o utilizador:
* **Strict-Transport-Security (HSTS)** - instrui o browser a usar sempre HTTPS. Sem isto, num Wi-Fi público, alguém poderia intercetar a ligação
* **Content-Security-Policy (CSP)** - restringe quais scripts o browser pode executar. Sem isto, a aplicação fica mais exposta a injeção de código malicioso
* **X-Frame-Options** - impede que o site seja carregado dentro de um *iframe* noutro site. Sem isto, um site malicioso poderia sobrepor botões falsos sobre o site real
* **X-Content-Type-Options** - previne que o browser interprete ficheiros de forma diferente do declarado
* **Referrer-Policy** - controla que informação é partilhada quando o utilizador navega para outros sites


* **Recomendação (sugerida por IA):** Configurar estes *headers* na plataforma de hosting. O Firebase Hosting permite configuração via firebase.json. São alterações de configuração simples que melhoram significativamente a postura de segurança.


---

### Página 6

**Relatório de Segurança [EMPRESA REDACTADA]**

**Observações (a confirmar)**

As seguintes observações são situações que identifiquei mas que não pude confirmar na totalidade. Incluo-as para conhecimento da equipa técnica, que estará em melhor posição para avaliar.

**01 - Persistência de sessão**

* **O que observei:** Depois de fazer login, se fechar o browser e o reabrir, o URL mostra a página de login. No entanto, se alterar manualmente o URL para a página do *dashboard*, a sessão ainda está ativa e consigo aceder ao *dashboard* sem fazer login novamente.
Ao inspecionar o IndexedDB (F12, separador Application), confirmei que o Firebase armazena localmente o accessToken (JWT), o refresh Token, e dados do perfil do utilizador. Enquanto o refresh Token estiver presente, a sessão renova-se automaticamente sem necessidade de nova autenticação.

*[Imagem redactada para proteção da entidade]*

*Screenshot do IndexedDB com tokens de sessão accessToken, refresh Token e dados de perfil redactados*

* **Ressalva:** Este é o comportamento padrão do Firebase Authentication, e pode ser intencional. No entanto, significa que qualquer pessoa com acesso ao computador/browser pode aceder à conta sem conhecer a password.

**02 - Possível falha na confirmação de pagamento MBWay**

* **O que observei:** Nas últimas vezes que fiz reservas, quando aceitei o pagamento MBWay na app dentro de poucos segundos, a reserva ficou registada corretamente e recebi email de confirmação. No entanto, quando demorei cerca de 2 minutos a aceitar o pagamento, o seguinte aconteceu:
* O dinheiro foi debitado da minha conta bancária
* A reserva não ficou marcada no sistema
* Não recebi email de confirmação
* O card de "aguardar pagamento" na aplicação ficou em loop, sem desaparecer
* Tive de contactar telefonicamente para regularização manual


* **Ressalva:** Isto aconteceu-me duas vezes, mas pode ser coincidência ou ter outra causa (ex: problema pontual de rede). A equipa técnica estará em melhor posição para avaliar. Tentarei replicar a situação nas próximas reservas.
* **Recomendação (sugerida por IA):** Se o problema se confirmar, a implementação de um *webhook* (callback) *server-side* garantiria que o pagamento é registado independentemente do estado do *frontend*. A nova versão da API (mencionada em F4) poderá já resolver este cenário.


---

### Página 7

**Relatório de Segurança [EMPRESA REDACTADA]**

**03 - Relógio do dispositivo de controlo de acesso dessincronizado**

* **O que observei:** O dispositivo de controlo de acesso (entrada do espaço) apresenta o relógio com uma diferença de aproximadamente 30 minutos em relação à hora real. No momento em que tirei a fotografia (19:32), o dispositivo mostrava 20:08. Tendo em conta que observei um PIN quando analisei a Consola no DevTools, tentei usar o PIN da minha reserva para abrir a porta, no entanto, o resultado foi sem efeito.
* **Porque pode ser relevante:** Um relógio dessincronizado pode afetar *logs* de acesso, dificultar auditorias, e indica que o dispositivo pode não estar configurado com sincronização automática de hora (NTP).


---

### Página 8

**Relatório de Segurança [EMPRESA REDACTADA]**

**Metodologia**

As vulnerabilidades foram identificadas através de duas abordagens complementares:

* **Observação manual:** durante a utilização normal da aplicação, com o painel F12 aberto, observei diretamente a exposição de dados na consola, as credenciais no URL de controlo de acesso, e as chaves de pagamento no código fonte
* **Análise automatizada:** utilizei um script *python* (desenvolvido com assistência de IA) que verificou *headers* de segurança HTTP, analisou o código fonte (main.dart.js) para identificar chaves e endpoints expostos, e testou a configuração das *Firestore Security Rules*

Em nenhum momento tentei aceder a dados de outros utilizadores de forma intencional, explorar vulnerabilidades, ou interferir com o funcionamento normal da aplicação.

**Nota sobre redação:** Este documento é uma versão redactada do relatório original. Dados identificadores da entidade (nome, dominio, logótipo, atividade), credenciais, chaves de API, *tokens* de sessão e dados pessoais de terceiros foram intencionalmente removidos ou mascarados. O relatório original foi entregue à entidade em questão no âmbito de um processo de *responsible disclosure*.

**Nota Final**

Partilho este relatório de boa fé, com o objetivo de contribuir para a melhoria da segurança da plataforma. Enquanto cliente, valorizo o serviço prestado e acredito que estas correções beneficiarão todos os utilizadores.

Fico disponível para esclarecer qualquer dúvida sobre as observações aqui descritas.

[REDACTADO]

[REDACTADO]

---
