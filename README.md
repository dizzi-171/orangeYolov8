# VISÃO COMPUTACIONAL ORANGEPIZERO2W+EV3
# COLOCAR PRINTS DE EXEMPLO EM TUDO

Tutorial de como instalar armbian e preparar o Orange Pi Zero 2w para projeto de robótica competitiva (Robocup Rescue Line - Equipe Ligeirinho) com Visão Computacional.

Versão do armbian utilizada é baseada em debian 12 (https://www.armbian.com/orange-pi-zero-2w/). Atualmente o projeto utiliza a versão Armbian 24.2.6 Bookworm.

Agora devemos instalar no cartão SD, dar boot e configurar inicialmente a instalação.

Atualize o APT (`sudo apt update`).  
Instale o python venv (`sudo apt install python3-venv`).  
Instale o dos2unix (`sudo apt install dos2unix`).  
Rode o comando `dos2unix` para converter a formatação do arquivo `setup_yolo.sh` de windows para linux.

Devemos executar o script `setup_yolo.sh` (`bash ./setup_yolo.sh`) para que as depencências sejam instaladas e o `venv` (ambiente virtual python) seja criado automaticamente.  
Caso queira fazer as instalações por conta própria, as dependências estão lsitadas no arquivo `requirements.txt`.
 
## Para utilizar duas câmeras (resgate e seguidor)

- Vamos precisar alterar o Device Tree do Orange.
- Primeiro, iremos alterar a porta USB que está como OTG para HOST:
  - Rode o comando `armbian-config` (`sudo armbian-config`), entrar em **System > dtc**.
  - O dts será gerado, encontre as portas USB.
  - Agora altere em todas as portas USB de `status = "disabled"` para `status = "okay"`, exceto na que possui `dr_mode = "peripheral"`.
    
  ![image](https://github.com/user-attachments/assets/aede3413-06be-4bf0-abaa-e6ffe0465555)

  - Altere nessa `peripheral`, de `status = "okay"` para `status = "disabled"`
  
  ![image](https://github.com/user-attachments/assets/39141d5b-bc1f-4e0a-936d-2c6889973348)

- Reinicie o Orange e confira se consegue se conectar nas duas câmeras agora.
- Lembre-se que para conectar, são criadas duas portas simbólicas `video` para cada porta USB física. Teste todas para ver se ambas conectam em alguma porta `video`.
- Com isso das portas simbólicas, ainda há o problema de não sabermos qual câmera é qual. Afinal, a câmera na USB1 pode estar em qualquer porta simbólica `video`.
- Para resolver esse problema, devemos garantir que estamos conectando na `video` correspondente à USB física correta:
  - Primeiro alteraremos o dts para ativar as portas sempre no mesmo modo.
    - Gere o dts novamente e nele encontre as portas USB físicas mais uma vez.
    - Perceba que cada porta USB física possui duas nomeações: OHCI e EHCI. OHCI significa USB 1.0 e EHCI, USB 2.0.
    - A alteração que faremos é desabilitar todas as OHCI, para garantir sempre a maior velocidade de comunicação disponível, além de evitar que uma porta seja identificada como 1.0 inesperadamente, o que pode causar problemas.
  
  - Reinicie o Orange, conecte uma câmera em uma porta USB física e rode o comando `ls /dev` para exibir as portas simbólicas `video` existentes.
    
    ![image](https://github.com/user-attachments/assets/45ec750d-beb6-4f3c-892c-d2e700f0809a)

  - Agora iremos analisar qual nome cada porta física recebe:
    - Rode o comando `readlink -f /sys/class/video4linux/video1/device` (Altere `video1` para cada porta `video` disponível).
    - A saída será algo como:
      
      ![image](https://github.com/user-attachments/assets/45f73666-e24d-4ede-bc23-cf068e82ab2b)

    - Note que temos o número `5200000`, que corresponde ao nome de uma das portas físicas no dts. Assim, essa porta USB corresponde à `5200000` no dts.
    - Lembre-se de ver o nome em cada porta `video`.
    - Logo, nós devemos, a cada execução, encontrar qual porta `video` tem link com as portas físicas que possuem no seu endereço, `520000`.
    - Faça o mesmo para a outra porta e assim saberá a numeração de cada porta USB.
  - Agora no código, precisaremos usar uma função que identifica qual porta `video` está conectada à porta USB física que determinada câmera está. Assim, relacionamos o objeto da câmera no código com a câmera correta.
  - Um exemplo de função para isso está no arquivo `encontrarPortaCamera.py`.

## Opcionais, mas altamente recomendados:

- Para utilizar comunicação Serial, você deverá alterar algumas configuraçõoes no Device Tree do Orange.
  - Primeiro confira quais portas são listadas no comando (`dmesg | grep tty`) e escolha a que irá usar.
  - Rode o comando `armbian-config` (`sudo armbian-config`), entrar em **System > dtc**.
  - Nesse arquivo encontre a uart pin que será utilizada, no nosso caso a uart5, copie o valor do `phandle` dessa porta, mais à frente o utilizaremos.

  ![image](https://github.com/user-attachments/assets/48b43938-7ec4-4295-81db-5badfeefa6bb)

  - Agora encontre a serial correspondente à essa uart, a ordem das seriais é 0,1,2,3..

  ![image](https://github.com/user-attachments/assets/041c8b21-5634-42e4-a59e-b0be800af591)

  - Nela, faça as seguintes alterações:
    - troque `status = 'disabled'` para `status = 'okay'`;
    - adicione abaixo dessa linha, as seguintes linhas:
      - `pinctrl-names = 'default';`
      - `pinctrl-0 = <0x55>` (no lugar de `0x55`, coloque o valor do `phandle` da uart que irá utilizar);
  - Reinicie o orange e verifique se a porta está ativa (`dmesg | grep tty`)

- Para subir código pelo VSCode somente apertando F5:
  - Deveremos primeiro permitir que nosso computador se conecte no orange via ssh sem senha.
    - Gere uma chave pública no seu computador, se já fez isso antes, não é necessário refazer (`ssh-keygen -t rsa -b 4096`).
    - `ssh-copy-id USUARIO@IP` (substitua o usuário e o IP pelo correto) (no linux).
    - No Windows não tem como ser feito com comando, abra o arquivo `id_rsa.pub`, na pasta `.ssh` dentro da pasta seu usuário, copie a linha.
    - Logue na placa por ssh e rode os comandos:
      - `mkdir -p ~/.ssh`
      - `nano ~/.ssh/authorized_keys`
      - Agora, copie o valor do arquivo `id_rsa.pub` e cole dentro desse arquivo, salve e saia.
      - Agora, qualquer login por ssh não pedirá mais senha.
  
  - Por fim, na pasta do projeto no VSCode, crie um arquivo `envia.bat`.
    - O código do que colocar no `envia.bat` está no arquivo `envia.bat` na pasta do projeto, lembre-se de substituir o que for necessário.
    - Você pode alterar como quiser, coloque o que achar necessário a ser feito quando apertar F5, como mais ou menos `scp`s, ou rodar o código nesse momento.

- Desabilitar o bluetooth para liberar CPU. (opcional)
  - `sudo systemctl stop bluetooth`
  - `sudo systemctl disable bluetooth`

- Dê permissão para que o shutdown e outros comandos necessários sejam executados sem precisar de senha: 
  - Digite `sudo visudo`
  - Dentro do arquivo, no final dele, coloque as seguintes linhas:
    - `orangepi ALL=(ALL) NOPASSWD: /usr/bin/psd-overlay-helper`
    - `orangepi ALL=(ALL) NOPASSWD: /sbin/shutdown`
    - `orangepi ALL=(ALL) NOPASSWD: /sbin/ifconfig` (troque `orangepi` pelo usuário escolhido)
  - Reinicie a placa e veja se não será mais pedido senha ao utilizar comandos.
 
- Para o código iniciar no momento que ligar, e quando parar de executar, ser executado novamente para permanecer em funcionamento, iremos criar um serviço.
  - Dentro da placa, crie um arquivo `start.sh` para ser executável, com o conteúdo:
    - O código do que colocar no `start.sh` está no arquivo `start.sh` na pasta do projeto, lembre-se de substituir o que for necessário. **# COLOCAR ARQUIVO START.SH AQUI**
  - Crie um serviço para iniciar o script toda vez que a placa ligar (`sudo nano /etc/systemd/system/<nome que quiser>.service`)
    - O código do que colocar no serviço está no arquivo `exemplo.service` na pasta do projeto, lembre-se de substituir o que for necessário.
  - Rode agora `sudo systemctl daemon-reexec`
  - Depois, `sudo systemctl enable <nome que deseja no serviço>.service`
  - Agora reinicie o orange e rode `ps aux`, dentro da lista enorme que aparece, o arquivo `start.sh` deve estar listado.
  - Você também pode visualizar os prints em tempo quase real (varia com a qualidade da rede), através do comando:  
    `sudo journalctl -u <nome do serviço> -f` 
