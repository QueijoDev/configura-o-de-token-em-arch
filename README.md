# Configuração Múltipla de Tokens Criptográficos no CachyOS / Arch Linux

**Objetivo:** Configurar e utilizar múltiplos tokens criptográficos (SafeNet eToken 5100, Feitian ePass2003 e Token GD/SafeSign) no mesmo sistema operacional, mitigando conflitos de hardware ao isolar os drivers em navegadores diferentes (Firefox e família Chromium).

---

## Fase 1: Preparação do Sistema e Serviços Base

Para que o Linux consiga se comunicar com as portas USB e ler os smartcards, os serviços essenciais devem estar instalados e rodando.

1. **Instale o pacote base de comunicação de smartcards (CCID):**
   ```bash
   sudo pacman -S ccid
   ```
2. **Ative e inicie o daemon do PC/SC:**
   ```bash
   sudo systemctl enable --now pcscd.socket
   sudo systemctl start pcscd.service
   ```
3. *(Opcional)* **Teste a leitura física:**
   Conecte um token e rode o comando `pcsc_scan`. Se aparecer "Card inserted", o hardware está comunicando. **Atenção:** Pressione `Ctrl+C` para sair, pois deixar este comando rodando "sequestra" o token e impede os navegadores de usá-lo.

---

## Fase 2: Instalação e Permissões dos Drivers (Módulos PKCS#11)

Cada fabricante exige seu próprio middleware ou biblioteca `.so` instalada no sistema (geralmente localizadas em `/usr/lib/`).

*   **SafeNet 5100:** Requer o pacote via AUR (`paru -S safenetauthenticationclient`). Arquivo: `/usr/lib/libeToken.so`
*   **ePass2003:** Arquivo: `/usr/lib/libcastle.so` (ou `libeps2003csp11.so`)
*   **Token GD (SafeSign):** Arquivo: `/usr/lib/libaetpkss.so.3`

**Resolução de Bloqueio (Passo Crítico):** 
Os navegadores recusam silenciosamente o carregamento de bibliotecas que não possuam permissão de execução, deixando o status do dispositivo como "Ausente". É obrigatório corrigir as permissões dos arquivos:

```bash
sudo chmod 755 /usr/lib/libeToken.so
sudo chmod 755 /usr/lib/libcastle.so*
sudo chmod 755 /usr/lib/libaetpkss.so*
```

---

## Fase 3: Configuração no Firefox (Exemplo: ePass2003)

Para evitar conflitos de leitura cruzada, dedicaremos o Firefox exclusivamente para um dos tokens.

1. Abra o Firefox e acesse **Configurações > Privacidade e Segurança**.
2. Role até o final da página e clique em **Dispositivos de Segurança...**.
3. Remova quaisquer módulos de outros tokens que estejam listados (clique neles e em **Descarregar**).
4. Clique em **Carregar**.
5. Preencha o Nome do Módulo (ex: `Feitian ePass2003`).
6. Em Nome do arquivo do módulo, insira o caminho absoluto:
   `/usr/lib/libcastle.so` (ou o arquivo correspondente).
7. Clique em OK. O status deverá confirmar a presença do hardware e o botão "Entrar" será liberado.

---

## Fase 4: Configuração em Navegadores Chromium (Brave / Chrome / Edge)

Navegadores baseados no motor Chromium não possuem interface gráfica para gerenciar módulos PKCS#11 no Linux. Eles utilizam o banco de dados NSS (`nssdb`) oculto no diretório do usuário. Dedicaremos estes navegadores ao **SafeNet**.

1. **Instale o pacote de ferramentas do NSS:**
   ```bash
   sudo pacman -S nss
   ```
2. **Crie a estrutura de diretórios do banco de dados de certificados:**
   ```bash
   mkdir -p ~/.pki/nssdb
   ```
3. **Inicialize o banco de dados:**
   ```bash
   certutil -d sql:$HOME/.pki/nssdb -N
   ```
   *Nota: O terminal pedirá uma senha para o banco de dados. Pressione `Enter` duas vezes para deixar em branco (isso evita que o navegador exija uma senha extra toda vez que for aberto).*

4. **Injete o driver do SafeNet no banco de dados do Chromium:**
   Com o navegador **fechado**, execute:
   ```bash
   modutil -dbdir sql:$HOME/.pki/nssdb/ -add "SafeNet" -libfile /usr/lib/libeToken.so
   ```
   *Pressione `Enter` para confirmar se um aviso (WARNING) aparecer na tela.*

5. **Validação:**
   Para confirmar que a injeção deu certo, liste os módulos instalados:
   ```bash
   modutil -dbdir sql:$HOME/.pki/nssdb/ -list
   ```
   O "SafeNet" deverá aparecer na lista.

---

**Conclusão:** A partir de agora, ao plugar o ePass2003, o fluxo deve ser feito via Firefox. Ao plugar o SafeNet, o fluxo deve ser feito via Brave/Chrome, garantindo estabilidade e zero conflito de drivers no ambiente de trabalho.
