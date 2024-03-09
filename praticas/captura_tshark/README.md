## **Exercício Prático sobre Captura e Análise de Tráfego**

## 1. Introdução

Neste exercício usaremos ferramentas de captura de tráfego e faremos extração de alguns dados com base em filtros e alguns scripts simples para selecionar, calcular e imprimir informações.

Adotaremos o **tshark**, parte do pacote de ferramentas do Wireshark. Apesar de provavelmente funcionarem em qualquer Linux e no Windows, os comandos foram testados no Ubuntu 22.04.

Referências:

- Opções do comando tshar

[https://www.wireshark.org/docs/man-pages/tshark.html](https://www.wireshark.org/docs/man-pages/tshark.html)

- Exemplos interessantes do tshark (algumas opções obsoletas, mas ainda interessante)

[https://www.krazyworks.com/practical-tshark-capture-filters/](https://www.krazyworks.com/practical-tshark-capture-filters/)

- Referência para campos disponíveis por protocolo

[https://www.wireshark.org/docs/dfref](https://www.wireshark.org/docs/dfref/t/tcp.html)

[https://www.wireshark.org/docs/dfref/t/tcp.html](https://www.wireshark.org/docs/dfref/t/tcp.html)

[http://packetlife.net/media/library/13/Wireshark_Display_Filters.pdf](http://packetlife.net/media/library/13/Wireshark_Display_Filters.pdf)


## 2. Preparativos
    
2.1. Faça login no ubuntu e mantenha-se logado no seu usuário, devidamente cadastrado como **sudoer** (não use "root")

2.2. Vamos Instalar o **Tshark**

```markdown
sudo apt -y install wireshark-common tshark tcpdump
sudo apt -y --fix-broken install
```

2.3. Alguns ajustes para permissão de captura para outros usuários

Criaremos um grupo "tshark", adicionaremos o usuário corrente ao grupo, ajustaremos as permissões para que o grupo possa executar o tshark.

```markdown
sudo groupadd tshark
sudo usermod -a -G tshark $USER
sudo chgrp tshark /usr/bin/dumpcap
sudo chmod 750 /usr/bin/dumpcap
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/dumpcap
sudo getcap /usr/bin/dumpcap
sudo dpkg-reconfigure wireshark-common #Selecione "Yes"
sudo chmod +x /usr/bin/dumpcap
# Atenção: faça login novamente para que o 
# usuário $USER entre no grupo criado. 
```

2.4. Para ver as interfaces de rede (a saída varia com seu computador)

```markdown
sudo tshark -D
1. enp3s0
2. any
3. lo (Loopback)
...
...
12. sshdump (SSH remote capture)
13. udpdump (UDP Listener remote capture)
```

2.5. Vamos verificar se o usuário comum (sem "sudo") consegue capturar tráfego com o Tshark (vide opção -D para identificar a interface "1"). Se não houver captura (indicada pelo contador de pacotes na saida), use o comando sempre precedido por "sudo" (ex: sudo tshark …)  -- **substitua o login abaixo (ifpb) pelo usuário adequado**

```markdown
tshark -i 1 -c 1 -q
# o esperado é algo assim:
# Capturing on 'enp4s0'
# 1 packet captured
```

2.6. Copiando alguns scripts AWK para cálculo do intervalo entre pacotes e contagem do número de ocorrências por intervalo de tempo

```markdown
wget https://tinyurl.com/denio-interval -O packet-interval.awk
wget https://tinyurl.com/denio-count -O session-count.awk 
# veja se deu certo:
ls -la *awk
```

Baixar também os dois arquivos:

https://www.dropbox.com/scl/fi/h160qhid4hab01mr6t9ow/curl-list-urls.sh?rlkey=gj6nepdm7mtjbfnt1btw8y2v0&dl=0

https://www.dropbox.com/scl/fi/gh1pqw130kkgi93w1xtea/lista-urls.txt?rlkey=h9e5j6fb4se0klzvkzfeko8pk&dl=0

## 3. Capturando tráfego

3.1. Capturando todo o tráfego da interface **#1** (vide opção -D para identificar a interface "1") e gravando a saída em um arquivo. Opção -**c N** indica condição de parada quando **N** pacotes forem capturados.

Depois que iniciar o comando, abra o navegador e navegue por várias páginas web diferentes. **Acesse sites https e também http**. A ideia é capturar 30 mil pacotes. Só siga para os próximos itens quando o comando terminar e voltar para o prompt de comando, indicando que todos os pacotes foram capturados.

```markdown
FILE=/tmp/capture.cap 
tshark -i 1 -w $FILE -c 30000
```

3.2. Obtendo informações sobre o arquivo de captura gerado

```markdown
sudo chmod a+r $FILE
sudo capinfos $FILE 
File name:           /tmp/capture.cap
File type:           Wireshark/... - pcapng
File encapsulation:  Ethernet
File timestamp precision:  nanoseconds (9)
Packet size limit:   file hdr: (not set)
Number of packets:   30 k
File size:           25 MB
Data size:           23 MB
Capture duration:    202,194530789 seconds
First packet time:   2021-05-06 10:38:11,505756408
Last packet time:    2021-05-06 10:41:33,700287197
Data byte rate:      118 kBps
Data bit rate:       949 kbps
Average packet size: 799,73 bytes
Average packet rate: 148 packets/s
SHA256:              76813345e3fce67dcc1cf9bce18de1c4e6e0fc8f29922ce2e40496a0358af554
RIPEMD160:           6ce0c954d036dd3886bc99ad7878290e282136ee
SHA1:                1a9cc3fd368bc02831a581dfc3980d60417cd6dc
Strict time order:   False
Capture hardware:    AMD Ryzen 7 1700 Eight-Core Processor           (with SSE4.2)
Capture oper-sys:    Linux 5.8.0-45-generic
Capture application: Dumpcap (Wireshark) 3.2.7 (Git v3.2.7 packaged as 3.2.7-1)
Number of interfaces in file: 1
Interface #0 info:
    **<<outras informações foram cortadas aqui>>**
```

## 4. Extraindo informações no nível TCP
    
4.1. Identificando Pacotes por Fluxos TCP e gerando um arquivo CVS com: 1)tempo relativo da chegada do pacote; 2) Protocolo (6=TCP); 3) IP de origem; 4) IP destino; 5) Porta de origem; 6) porta destino; 6) tempo entre chegadas (inter arrival time)

```markdown
tshark -r ${FILE} -Y tcp -T fields -e frame.time_relative -e ip.proto -e ip.src -e ip.dst -e tcp.srcport -e tcp.dstport -e ip.len -e frame.time_delta -E separator=,
```

4.2. Selecionando apenas pacotes que envolvem solicitação de conexão TCP (Flags SYN ativado sem ACK) - observe que a saída é gravada no arquivo **tcp.tmp**

```markdown
tshark -r ${FILE} -Y '(tcp && tcp.flags.syn==1 && tcp.flags.ack==0)' -T fields -e frame.number -e frame.time_relative -e ip.proto -e ip.src -e ip.dst -e tcp.srcport -e tcp.dstport -e ip.len -e frame.time_delta -E separator=, > tcp.tmp
```

4.3. Processando pacotes que envolvem solicitação de conexão TCP para calcular o tempo entre **conexões**

```markdown
awk -F , -f ./packet-interval.awk < tcp.tmp > tcp-session-init.csv
```

4.4. Contando ocorrências de solicitações de conexão TCP por intervalo de tempo

```markdown
awk -F , -v slot=10 -v sep="\t" -f ./session-count.awk < tcp-session-init.csv > tcp-session-count.csv
```

**Comentário**
A saída gerada pode ser usada para construir um histograma. Na saída abaixo, temos: "no intervalo entre 0 e 60 segundos, tivemos 81 conexões; no intervalo entre 60 e 120 segundos, tivemos 19 conexões" ...

```markdown
seq,timea,timeb,count
1,0,10,81
2,10,20,19
3,20,30,65
4,30,40,16
```

Dependendo da quantidade de respostas no arquivo <code>tcp-session-count.csv</code></strong> (poucas linhas), você precisará ajustar o parâmetro **slot** para exibição de mais resultados.

4.5. Obtendo estatísticas de volume de dados e duração de Fluxos TCP (o Tshark chama fluxos de "conversations")

```markdown
tshark -r ${FILE} -q -z 'conv,tcp,ip'
```

**Comentário**
O comando acima gera uma saída com as seguintes colunas:

1. `srcip:srcport `
2. `<-> (string fixo)`
3. `dstip:dstport `
4. `número de frames no sentido dst`🡪`src`
5. `volume de bytes no sentido dst`🡪`src`
6. `número de frames no sentido dst`🡪`src`
7. `volume de bytes no sentido dst`🡪`src`
8. `total de frames nos dois sentidos`
9. `total de bytes nos dois sentidos`
10. `tempo relativo do início da sessão TP`
11. `duração da sessão TCP`

4.6. O comando abaixo extrai colunas selecionadas (1, 3, 8 e 9) das informações das sessões

```markdown
tshark -r ${FILE} -q -z 'conv,tcp,ip' | grep '<->' | awk '{ print $1 "," $3 "," $8 "," $9 "," $11 }' 
```

## 5. Extraindo informações HTTP

5.1.  Listando tráfego HTTP (*request only*)

```markdown
tshark -r ${FILE} -Y 'tcp && (tcp.dstport==80 || tcp.dstport==443)'  -T fields -e frame.number -e frame.time_relative -e ip.proto -e ip.src -e ip.dst -e tcp.srcport -e tcp.dstport -e ip.len -e frame.time_delta -E separator=\, > http.tmp

awk -F , -f ./packet-interval.awk < http.tmp > http-request.csv
awk -F , -v slot=10 -v sep="\t" -f ./session-count.awk < http-request.csv > http-session-count.csv
```

5.2. Listando  todos os domínios visitados

```markdown
tshark -r ${FILE} -Y http.request -T fields -e http.host | sort -u
```

5.3. Listando os N domínios mais visitados (N=3)

```markdown
tshark -r "$FILE" -Y http.request -T fields -e http.host -e http.request.uri | sed -e 's/?.*$//' | sed -e 's#^\(.*\)\t\(.*\)$#http://\1\2#' | sort | uniq -c | sort -nr | head -n 3
```

5.4. Observando os dados das conexões HTTP inseguras (porta 80)

```markdown
for stream in $(tshark -nlr "${FILE}" -Y '(tcp.flags.syn==1 && tcp.dstport==80)' -T fields -e tcp.stream | sort -n | uniq) ; \
do echo "DADOS DO FLUXO $stream" ; tshark -nlr "${FILE}" -q -z "follow,tcp,ascii,$stream" ; \
done | more
```


## 6. Extraindo informações DNS

6.1. Listando tráfego DNS (query & response). Mostra colunas: 1) número do pacote na sequência de captura; 2) tempo de chegada (relativo); 3) protocolo (UDP=17); 4) IP origem; 5) IP destino; 6) tamanho do pacote; 7) intervalo entre pacotes (inter arrival time)

```markdown
tshark -r ${FILE} -Y dns -T fields -e frame.number -e frame.time_relative -e ip.proto -e ip.src -e ip.dst -e udp.srcport -e udp.dstport -e ip.len -e frame.time_delta | more
```

6.2.  Perguntas DNS mais frequentes

```markdown
tshark -r "${FILE}" -T fields -e dns.qry.name -Y "dns.flags.response eq 0" | sort | uniq -c | sort -rn | head -n 10
```

6.3. Listando tráfego DNS (query only), calculando intervalo entre consultas e ocorrências por bloco de tempo

```markdown
tshark -r ${FILE} -Y 'dns && udp.dstport==53' -T fields -e frame.number -e frame.time_relative -e ip.proto -e ip.src -e ip.dst -e udp.srcport -e udp.dstport -e ip.len -e frame.time_delta -E separator=\, > dns.tmp

awk -F , -f ./packet-interval.awk < dns.tmp > dns-query.csv
awk -F , -v slot=10 -v sep="\t" -f ./session-count.awk < dns-query.csv> dns-query-count.csv
```

**O QUE DEVE SER ENVIADO PARA AVALIAÇÃO DO PROFESSOR:**

Você deve enviar como resposta deste exercício os arquivos: <code>tcp-session-count.csv</code></strong> obtido no item 4.4, <strong><code>http-session-count.csv</code></strong> obtido no item 5.1 e <strong><code>dns-query-count.csv</code></strong> obtido no item 6.3.