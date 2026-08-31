# Configuração Múltipla de Tokens Criptográficos no CachyOS / Arch Linux

**Objetivo:** Configurar e utilizar múltiplos tokens criptográficos (SafeNet eToken 5100, Feitian ePass2003 e Token GD/SafeSign) no mesmo sistema operacional, mitigando conflitos de hardware ao isolar os drivers proprietários em navegadores diferentes (Firefox e família Chromium).

---

## Fase 1: Preparação do Sistema e Serviços Base

Para que o Linux consiga se comunicar com as portas USB e ler os smartcards, os serviços essenciais devem estar instalados e rodando.

1. **Instale os pacotes base de comunicação de smartcards:**
   ```bash
   sudo pacman -S ccid pcsc-tools
   ```
2. **Ative e inicie o daemon do PC/SC:**
   ```bash
   sudo systemctl enable --now pcscd.service
   sudo systemctl enable --now pcscd.socket
   ```
3. *(Opcional)* **Teste a leitura física:**
   Conecte um token e rode o comando `pcsc_scan`. Se o nome do token e a mensagem "Card inserted" aparecerem, o hardware está comunicando. 
   ⚠️ **Aviso Crítico:** Pressione `Ctrl + C` para sair do `pcsc_scan` logo após o teste. Deixar este comando rodando "sequestra" o token no terminal e causa o status de "Ausente" nos navegadores.

---

## Fase 2: Instalação e Permissões dos Drivers (Módulos PKCS#11)

Não utilize pacotes genéricos como o `opensc` para mapear os tokens, pois ele tenta ler todos os dispositivos simultaneamente, gerando conflito (status "Ausente"). Use os drivers proprietários via AUR:

1. **Instale os pacotes correspondentes aos seus tokens:**
   * **SafeNet 5100:** `paru -S sac-gui` (ou `safenet-authentication-client`)
   * **ePass2003:** `paru -S epass2003-sdk-linux`
   * **Token GD (SafeSign):** `paru -S safesignidentityclient`

2. **Descubra o caminho exato do driver instalado:**
   O Arch Linux pode mudar o nome ou a pasta do arquivo `.so`. Para ter certeza do caminho, rode:
   ```bash
   pacman -Ql epass2003-sdk-linux | grep "\.so"
   ```
   *(Substitua o nome do pacote caso esteja procurando o do SafeNet ou GD).*

3. **Resolução de Bloqueio (Passo Obrigatório):** 
   Os navegadores recusam silenciosamente bibliotecas sem permissão de execução. Aplique a permissão nos arquivos descobertos no passo anterior:
   ```bash
   sudo chmod 755 /usr/lib/libcastle.so.1.0.0  # Exemplo do ePass2003
   sudo chmod 755 /usr/lib/libeToken.so       # Exemplo do SafeNet
   sudo chmod 755 /usr/lib/libaetpkss.so*     # Exemplo do GD SafeSign
   ```

---

## Fase 3: Configuração no Firefox (Exclusivo para ePass2003)

Dedicaremos o Firefox exclusivamente para o ePass2003, usando seu driver proprietário para evitar leitura cruzada.

1. Abra o Firefox e acesse **Configurações > Privacidade e Segurança**.
2. Role até o final da página e clique em **Dispositivos de Segurança...**.
3. **Limpeza:** Se houver um módulo chamado `Novo módulo PKCS#11` ou `OpenSC` listando seus tokens com status "Ausente", selecione-o e clique em **Descarregar**.
4. Clique em **Carregar**.
5. **Nome do módulo:** `Feitian ePass2003` (ou nome de sua preferência).
6. **Nome do arquivo do módulo:** Insira o caminho absoluto validado na Fase 2 (ex: `/usr/lib/libcastle.so.1.0.0`).
7. Clique em OK. O status deverá confirmar a presença do hardware e o botão "Entrar" será liberado para digitar o PIN.

---

## Fase 4: Configuração no Brave / Chromium (Exclusivo para SafeNet)

Navegadores baseados no motor Chromium (Brave, Chrome, Edge) não possuem interface gráfica para gerenciar módulos PKCS#11 no Linux. Eles utilizam o banco de dados NSS (`nssdb`) oculto no diretório do usuário. 

1. **Instale o pacote de ferramentas do NSS:**
   ```bash
   sudo pacman -S nss
   ```
2. **Crie a estrutura e inicialize o banco de dados do navegador:**
   ```bash
   mkdir -p ~/.pki/nssdb
   certutil -d sql:$HOME/.pki/nssdb -N
   ```
   *Nota: O terminal pedirá uma nova senha. Pressione `Enter` duas vezes para deixar em branco (isso evita que o Brave exija uma senha extra ao abrir).*

3. **Injete o driver do SafeNet no banco de dados:**
   Com o navegador **totalmente fechado**, execute o comando apontando para o `.so` do SafeNet:
   ```bash
   modutil -dbdir sql:$HOME/.pki/nssdb/ -add "SafeNet" -libfile /usr/lib/libeToken.so
   ```
   *Pressione `y` ou `Enter` caso apareça um aviso (WARNING) confirmando a ação.*

4. **Validação:**
   Para confirmar que a injeção deu certo, liste os módulos instalados:
   ```bash
   modutil -dbdir sql:$HOME/.pki/nssdb/ -list
   ```
   O módulo "SafeNet" deverá aparecer listado no terminal.

**Conclusão:** O ambiente está isolado. Ao utilizar o ePass2003, acesse os portais via Firefox. Ao utilizar o SafeNet, acesse via Brave. Isso garante zero conflitos de drivers e estabilidade.
