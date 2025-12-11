# ponderada_geral# Análise estática do código ESP32 Web Server

## 1. Introdução
Este documento apresenta uma revisão da análise estática de um servidor web implementado em um ESP32, desenvolvido a partir do exemplo disponibilizado por Rui Santos (Random Nerd Tutorials). O código monta um servidor HTTP na porta 80 que permite acionar remotamente os GPIOs 26 e 27 por meio de requisições GET realizadas via navegador, sem qualquer camada adicional de proteção.

A revisão concentra-se na identificação das fragilidades do código, considerando aspectos de segurança, tratamento de entrada, exposição na rede e possíveis impactos sobre disponibilidade, integridade e confidencialidade. Com base nessas observações, são descritos cenários de ataque viáveis e propostas recomendações técnicas para mitigação.


## 2. Vulnerabilidades

### 2.1 Autenticação
Qualquer pessoa conectada à mesma rede (ou com rota até o dispositivo) consegue manipular os GPIOs simplesmente enviando requisições HTTP GET, sem precisar se autenticar.
### 2.2 Tráfego não Criptografado
O tráfego HTTP entre o cliente (navegador) e o ESP32 é não criptografado. Um atacante presente na mesma rede Wi-Fi, ou com acesso ao roteador, pode interceptar, ler e alterar as requisições enviadas ao servidor. Como o servidor aceita qualquer comando baseado apenas no caminho da URL, o invasor pode injetar comandos maliciosos, por exemplo alterando um pacote legítimo do usuário.


## 3. Ataques

### 3.1 Controle remoto não autorizado

#### 3.1.1 Passo-a-passo do ataque:
1. O atacante obtém acesso à mesma rede do ESP32.  
2. Descobre o IP do ESP32 (nmap, fing, ARP/MDNS, roteador etc.).  
3. Envia requisição HTTP GET para o endpoint de interesse.  
4. O atacante aciona GPIOs por comandos simples.

#### 3.1.2 Probabilidade de ocorrência
Alta em redes domésticas sem segmentação ou firewall. Média em redes corporativas mais restritas. Como o servidor não possui autenticação e usa HTTP simples, basta que o ESP32 seja alcançável.

#### 3.1.3 Impacto estimado
Alto — GPIOs podem controlar relés, motores, portas etc., causando danos físicos ou riscos operacionais. O impacto em privacidade varia conforme o uso.

#### 3.1.4 Risco resultante
Alto — combinação de alta probabilidade e alto impacto.

</br>

### 3.2 Tráfego HTTP sem criptografia

#### 3.2.1 Passo-a-passo do ataque:
1. O atacante identifica o ESP32 na rede.  
2. Posiciona-se entre o cliente e o ESP32 (ARP spoofing, rogue AP, roteador comprometido).  
3. Todo o tráfego passa pelo atacante.  
4. O invasor altera requisições ou respostas (ex.: trocar `/26/off` por `/26/on`, alterar HTML etc.).

#### 3.2.2 Probabilidade
A probabildiade é alta em redes domésticas e menor em redes corporativas bem gerenciadas.

#### 3.2.3 Impacto estimado
O impacto estimado é muito alto, pois permite leitura e alteração de comandos, controle furtivo dos GPIOs, modificação de interface, ocultação da presença do atacante etc.

#### 3.2.4 Risco resultante
O risco é muito alto já que a probabilidade razoável somada ao impacto crítico.

</br>

### 3.3 Cross-Site Request Forgery (CSRF)

Como o ESP32 aceita comandos via GET, qualquer página maliciosa pode forçar o navegador de um usuário da mesma rede a disparar ações contra o ESP32, sem que o usuário perceba.

#### 3.3.1 Passo-a-passo:
1. O atacante cria uma página com uma imagem invisível:
```html
<img src="http://<IP_DO_ESP>/26/on" style="display:none">
```
2. O usuário abre a página (rede interna).
3. O navegador envia automaticamente o GET para o ESP32.
4. O GPIO é acionado sem intenção do usuário.

### 3.3.2 Probabilidade
Esse ataque é de baixa probabilidade, ele requer que o usuário esteja na mesma rede do ESP32, que ele acesse uma página maliciosa, e não funciona se o usuário não usar navegador ou se o ESP32 estiver isolado.

### 3.3.3 Impacto
O impacto é de baixo a médio já que só funciona se o navegador do usuário enviar a requisição e não dá controle total ao atacante, apenas um clique induzido. Dessa forma, ele pode acionar GPIOs, mas não é persistente nem silencioso como MITM.

### 3.3.4 Risco resultante
Ele tem baixa probabilidade moderada e impacto limitado.

</br>


## 4. Conclusão da análise
Foi elaborada uma tabela consolidada dos ataques e ordenados os ataques desta tabela de forma decrescente (do maior risco para o menor risco).

| Título do Ataque                                      | Probabilidade | Impacto    | Risco       |
|-------------------------------------------------------|---------------|------------|-------------|
| Man-in-the-Middle (MITM) ativo, interceptação e modificação de tráfego |Alta    | Muito Alto | Muito Alto  |
| Controle Remoto Não Autorizado dos GPIOs              | Alta          | Alto       | Alto        |
| CSRF: Execução Indireta de Comandos via Navegador     | Baixa         | Baixo      | Baixo       |

## 4. Vídeo do hardware
https://drive.google.com/file/d/1OW4Em_-CA1642B4_aOaKBbBbo6LdLpka/view?usp=sharing 

Acessando IP de diferentes computadores, mostrando a falta de segurança: https://drive.google.com/file/d/1AEfa8cmIlqFp4XENeSZIpoR7lw8ESLNR/view 

O vídeo apresenta o acesso indevido a um dispositivo devido à ausência de mecanismos de segurança no servidor implementado no ESP32. Esse comportamento demonstra exatamente o tipo de vulnerabilidades analisadas neste relatório.
Para evidenciar essa vulnerabilidade, utilizou-se um segundo computador, separado do ambiente de desenvolvimento original, com o objetivo de simular um atacante externo.
Durante a demonstração, esse computador adicional consegue acessar diretamente as rotas expostas pelo servidor do ESP32 sem qualquer autenticação, criptografia ou controle de permissões. Isso evidencia que o dispositivo responde a requisições não autorizadas e não valida a origem das conexões, permitindo a visualização e manipulação dos dados do sistema.
