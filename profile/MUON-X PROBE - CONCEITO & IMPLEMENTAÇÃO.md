
---
<div style="display: flex; flex-direction: row;">
	<div style="flex: 1; text-align: left;">
		<p><strong>Documento desenvolvido por:</strong> <i>Eduardo Xavier Oliveira Sá - Colégio Internato dos Carvalhos</i></p>
	</div>
	<div style="flex:1; text-align: right;">
		<p>Abril 2025</p>
	</div>
</div>

## Detecção de Muões Cósmicos na superfície terrestre e na atmosfera

Tendo em conta a meta de, após um conjunto de observações à superfície, poder realizar uma missão de medição e detecção de muões cósmicos, a bordo de um balão meteorológico, ao nível da estratosfera, está a ser desenvolvido um protótipo de um dispositivo semelhante a uma sonda, que, albergando um conjunto de sensores de diferentes tipos, obtém dados relativos à contagem de deteções de partículas com origem em raios cósmicos, especialmente os muões, capazes de ionizar um tubo de Geiger-Muller no interior da sonda.


> **Nota:** Embora qualquer tipo de radiação suficientemente energética possa originar um evento de ionização no interior do tubo de Geiger-Müller, os muões representam, de acordo com os dados científicos, a maior percentagem de partículas ionizantes provenientes de radiação cósmica, na superfície terrestre.

---
## Conceito inicial
<div class="dualColumnLayout">
	<div>
		<p>Partindo do conceito demonstrado na reunião anterior, um detetor de Geiger-Muller (concebido no LIP) e um sensor de temperatura e pressão barométrica (BMP280) foram conectados ao microcontrolador <i>Raspberry Pi Pico W</i>, tendo sido devidamente programado para registar a contagem de muões detetados e monitorizar a temperatura e pressão do ambiente em que estas medições são realizadas.</p>
	</div>
	<img src="BASIC_MUON_COUNTER_CIRCUIT.png" style="width:30%"/>
</div>

O módulo Raspberry Pi é programado através do ambiente integrado de desenvolvimento (IDE) Arduino IDE, na liguagem de baixo nivel C++, permitindo um ótimo controlo de componentes externos ao módulo ligados, entre os quais o contador de Geiger-Muller e o sensor BMP280.

No sentido de testar este conceito básico e preliminar, foi utilizado o seguinte código, fornecido pelo LIP:

##### geiger_exemplo_5.ino (C/C++):

``` cpp
#include <Wire.h>
#include <SPI.h>
#include <Adafruit_BMP280.h>

#define GEIGER_INPUT 7
#define GEIGER_CONV_FACTOR 0.00812  // factor de conversao de CPM para uSv/h

#define GEIGER_GREEN_LED 15 // pin do led verde
#define GEIGER_YELLOW_LED 14 // pin do led amarelo
#define GEIGER_RED_LED 13 // pin do led vermelho
#define BMP280_CS 17 // pin que permite selecionar o sensor no bus de SPI

unsigned long contagens = 0; // variavel que guarda as contagens entre calculos

unsigned long last_read_micros = 0; // numero de microsegundos desde que o programa iniciou. Ao fim de ~70 min faz rollover (volta a zero)
unsigned long last_read_millis = 0; // numero de milissegundos desde que o programa começou.

unsigned long nivel_dose_baixa = 40;
unsigned long nivel_dose_alta  = 150;

Adafruit_BMP280 bmp(BMP280_CS);

bool bmp_conectado = false;

void count_pulse()
{
  if(last_read_micros + 2000 < micros() || last_read_millis + 2 < millis())
  {
    contagens++;

    last_read_micros = micros();
    last_read_millis = millis();
  }
} 

void setup()
{
  pinMode(GEIGER_GREEN_LED, OUTPUT);
  pinMode(GEIGER_YELLOW_LED, OUTPUT);
  pinMode(GEIGER_RED_LED, OUTPUT);
  pinMode(GEIGER_INPUT, INPUT);
  attachInterrupt(digitalPinToInterrupt(GEIGER_INPUT),count_pulse,FALLING);

  while (!Serial && millis() < 2000)
  {
    delay(10); // 10 ms
  }

  Serial.begin(9600);
  pinMode(BMP280_CS, OUTPUT);
  
  bmp_conectado = bmp.begin(0x76, 0x58);

  if (!bmp_conectado) {
    Serial.println("Sensor pressao temperatura não encontrado");
  }

  Serial.println("Serial started...setup done!");

}

void loop()
{
  unsigned long cpm = contagens * 6;

  if (cpm <= nivel_dose_baixa)
  {
    digitalWrite(GEIGER_GREEN_LED,HIGH);
    digitalWrite(GEIGER_YELLOW_LED,LOW);
    digitalWrite(GEIGER_RED_LED,LOW);
  }
  else if (cpm <= nivel_dose_alta)
  {
    digitalWrite(GEIGER_GREEN_LED,LOW);
    digitalWrite(GEIGER_YELLOW_LED,HIGH);
    digitalWrite(GEIGER_RED_LED,LOW);
  }
  else
  {
    digitalWrite(GEIGER_GREEN_LED,LOW);
    digitalWrite(GEIGER_YELLOW_LED,LOW);
    digitalWrite(GEIGER_RED_LED,HIGH);
  }

  contagens = 0;

  float dose = cpm * GEIGER_CONV_FACTOR;

  Serial.printf("Contagens por minuto : %u\n",cpm);
  Serial.printf("Dose num humano      : %f uSv/h\n",dose);

  if(bmp_conectado) {
    float temperatura = bmp.readTemperature();
    float pressao = bmp.readPressure();
    Serial.printf("Temperatura          : %f ºC\n",temperatura);
    Serial.printf("Pressão              : %f Pa\n",pressao);
  }

  delay(5000);
}
```

Este pequeno algoritmo permitiu a obtenção da **taxa de deteção de partículas por minuto*, valores de **dose de radiação ionizante** (em $\mu Sv/h$ - microsieverts por hora), **temperatura do ar** (ºC), e **pressão atmosférica** (Pa), gerando um output com o seguinte formato:

##### AMOSTRA (TXT):

```
-> Contagens por minuto: 90
-> Dose (num humano): 0.730800 uSv/h
-> Temperatura: 23.240000 ºC
-> Pressão: 101148.093750 Pa
```


Deste modo, este dispositivo já permite a detecção e contagem de muões cósmicos na superfície terrestre, no entanto, prescinde de um sistema mais completo e avançado de aquisição e análise de dados, dado que as medições são apenas reveladas na **consola *serial* (*Serial Monitor*)**. Para além deste fator, a eventual utilização do aparelho num cenário de voo requer um conjunto de outros sistemas de controlo e comunicação, capazes de não só enviar informação em tempo real para a equipa científica, como também de calcular e avaliar parâmetros de orientação, altitude, movimento e geolocalização, assegurando à sonda uma maior robustez e resiliência.

Neste sentido, foi idealizado um novo conceito de dispositivo de medição, especificamente desenhado tendo em consideração as condições encontradas numa missão de voo num balão meteorológico.

---
## X-MUON_PROBE:

### O novo conceito:

Além de efetuar medições de contagem de muões e de temperatura / pressão ambientes, a sonda possui um sistema de geo-orientação composto por um giroscópio/acelerómetro e um magnetómetro (bússola), assim como um módulo de comunicação rádio LoRa (Long Range), permitindo o envio dos dados de medição efetuados e de controlo de voo (orientação, altitude, temperatura no interior do dispositivo, integridade dos sistemas eletrónicos) para um receptor ativo à superfície terrestre, que, por sua vez, armazena os dados recolhidos de forma estruturada numa Base de Dados acessível via *Ethernet*, para posterior análise, através de uma interface web, alojada localmente, num computador.

### Construção e desenvolvimento da sonda:

#### Componentes utilizados

Tendo em conta o modelo inicialmente concebido, o **Raspberry Pi Pico W** foi, de novo, utilizado como microcontrolador central do sistema eletrónico da sonda, permitindo a compilação e execução de código desenvolvido em **C/C++**, no ambiente de desenvolvimento Arduino IDE, integrando as devidas bibliotecas de código necessárias para a comunicação com diversos componentes associados.

Dada a natureza experimental deste projeto, todos os componentes utiizados foram conectados através de uma placa de prototipagem (breadboard), oferecendo uma maior flexibilidade em termos de futuras alterações, quando comparada com a solda numa placa permanente.

**Deste modo, eis a lista completa de componentes integrados no dispositivo:**
- Raspberry Pi Pico W (microcontrolador);
- Detetor de Geiger-Muller (LIP);
- BMP280 (Sensor de temperatura e pressão atmosférica);
- MPU6050 (Acelerómetro e Giroscópio);
- QMC5883L (Magnetómetro / Bússola);
- EBYTE LoRa E32433T30D (Módulo de comunicação rádio de longa distância).

#### Circuito final:

Após o desenvolvimento deste dispositivo, foi obtido um modelo final representado pelo circuito esquematizado abaixo.

![[MUON-X_PROBE.png]]


<div style="display: flex; gap: 10px;">
<p>Embora não figure neste diagrama, foi utilizada a antena EBYTE TX433-JZ-5, do mesmo fabricante do módulo de comunicação rádio.</p>
<img src="./ASSETS/Pasted image 20260601003444.png" width="300" />
</div>

#### Código desenvolvido:

No que concerne ao código-fonte da sonda, este foi disponibilizado na plataforma online GitHub, possibilitando a colaboração de  futuros membros do projeto, no próximo ano letivo. O repositório é globalmente acessível através do seguinte link: https://github.com/eduard0sa/ATM_MUON_DETECTION_PROBE.

### Desenvolvimento do módulo de comunicação terrestre:

#### Componentes utilizados:

Para possibilitar a comunicação com a sonda, foi desenvolvido um módulo de comunicação terrestre, utilizando como micro-controlador o Arduino Leonardo. Utilizando um módulo de comunicação rádio do mesmo modelo do utilizado na sonda, este dispositivo também possui uma interface de rede (Ethernet Shield), que possibilita a conexão a um computador, numa entrada RJ45, onde está alojada a base de dados. Para efeitos de prototipagem, foi também utilizada uma Screw Shield, onde os fios são conectados em entradas fixas por parafusos (estes garantem uma maior estabilidade das ligações, não impossibilitando a alteração do circuito).

**De forma resumida, o desenvolvimento do receptor envolveu os componentes abaixo listados:**
- Arduino Leonardo (microcontrolador);
- Arduino Ethernet Shield (interface de rede);
- Screw Shield 1.0 (placa de prototipagem);
- EBYTE LoRa E32433T30D (Módulo de comunicação rádio de longa distância).

#### Circuito final:

Após o desenvolvimento deste dispositivo, foi obtido um modelo final representado pelo circuito esquematizado abaixo.

![[XMUON_GROUND_COMMS_DEVICE.png]]

Neste circuito, a Ethernet Shield e a Screw Shield são acopladas nos pinos do Arduino Leonardo, de forma a formarem um estrutura em pilha (daí a designação shield - escudo).

#### Código desenvolvido:

À semelhança da sonda, o código desenvolvido para este dispositivo receptor foi disponibilizado no seguinte repositório GitHub: https://github.com/X-MUON-PROBE/XMUON-GROUND-COMMS-MODULE/.

### Software de telemetria e visualização de dados da sonda:

Uma das principais características deste sistema é, efetivamente, o seu software de telemtria. Este possibilita a aquisição das medições periodicamente efetuadas em tempo real, que, após serem permanentemente guardados numa base de dados, são visualizados numa interface web, sob a forma de gráficos e outros tipos de estatísticas, assim como a monitorização do estado do dispositivo no decorrer da experiência.

Deste modo, será necessária a utilização de um computador onde possamos ligar o módulo de comunicação terrestre, e executar este software, que, assim que for dada a instrução de começo da experiência (na própria interface), iniciará o registo dos dados da sonda.

#### Arquitetura do software:

Esta solução de aquisição de dados foi conceptualizada de forma a integrar três sistemas distintos: **uma Base de Dados, um serviço API, e uma interface web**.

##### Base de Dados:

Para o armazenamento dos dados, esta base de dados foi implementada em PostgreSQL, de forma a ser facilmente transferível entre dispositivos, ou até eventualmente migrada para a cloud.

Esta armazena todos os registos gerados em cada medição efetuada pela sonda na tabela *telemetry_records*, associando-os a uma determinada experiência, identificada com um nome atribuído pelo utilizador e por um Id gerado pela própria base de dados. Os atributos das diversas experiências são registados na tabela *telemetry_missions*.

Visto que o ambiente gráfico de visualização de dados é atualizado em tempo real, existe também uma tabela (*dashboard_wss_connexions*) que armazena, temporariamente, todas as sessões de visualização ativas num determinado momento, associando cada sessão a um identificador gerado pela API, que estabelece uma comunicação de longa duração (websockets) com o cliente.

A estrutura da base de dados é esquematizada por este diagrama ER (tipo de esquema frequentemente utilizado durante a modelação teórica de bases de dados):

![[Pasted image 20260531220651.png]]

##### Serviço API:

Para a recepção dos dados enviados pelo dispositivo receptor, uma API (serviço backend de longa duração, que responde a pedidos enviados por uma aplicação frontend, como, por exemplo, o website do software), foi desenvolvida em ASP.NET CORE (C#), escutando todos os pacotes de informação da experiência, e armazenando a mesma na base de dados, de forma estruturada. Durante este processo de inserção de dados, são também geradas estatísticas mais complexas com base nos dados "crus" recebidos, visto que o tamanho dos pacotes de informação enviados pela sonda são limitados a um reduzido tamanho de 58 bytes.

![[Screenshot (144).png]]

Tanto o código da API como os scripts de criação da base de dados estão disponíveis no repositório Github com a seguinte hiperligação: https://github.com/X-MUON-PROBE/XMUON-TELEMETRY-API/.

##### Interface Web:

Finalmente, de modo a permitir a visualização dos dados de telemetria obtidos, é desenvolvida uma interface visual (página desenvolvida em REACT), acessível através do browser do PC em que o sistema é executado.
Através de gráficos e valores numéricos, são apresentados o estado atual do dispositivo (temperatura dos componentes, orientação, aceleração e rotação da sonda) e as estatísticas calculadas através dos dados obtidos, incluindo a progressão do número de contagens de partículas em função do tempo, dos registos efetuados, ou a variação de temperatura, pressão e altitude em função do tempo.

![[Pasted image 20260531220100.png]]


O código desenvolvido no âmbito desta página está disponivel no seguinte  repositório Github: https://github.com/X-MUON-PROBE/XMUON-TELEMETRY-OPS-PORTAL.


## Setup para deteção de muões cósmicos:

Após o desenvolvimento deste sistema integrado de deteção de muões cósmicos, resta testar a viabilidade do mesmo. De forma a simular um cenário de experiência real, a sonda foi ligada e o recetor conectado a um computador que corre o software de telemetria. Após estabelecer ligação entre o PC e o recetor, criando uma nova experiência na interface web, é possivel visualizar os dados obtidos no ecrã do computador, tal como representado na figura.

![[WhatsApp Image 2026-05-31 at 11.51.32 PM.jpeg]]


## Considerações a conhecer em eventual lançamento:

Ressalvo que o modelo aqui desenvolvido constitui um protótipo funcional, que, no entanto, não reúne todas as condições que possam surgir num cenário de voo. Desta forma, serão aqui apresentadas algumas considerações a ter na preparação do dispositivo para o lançamento:

#### Interferências eletromagnéticas no interior da sonda:

É conhecido o facto de a sonda possuir um sistema de geo-orientação constituído por uma bussola (magnetómetro) e um acelerómetro/giroscópio. No entanto, componentes como a bússola são bastante sensíveis a campos magnéticos gerados na sua periferia, entre os quais aqueles gerados na antena do módulo de comunicação radio, levando a que seja feitas leituras erradas dos valores do campo magnético terrestre (cerca de 28-65 μT), e assim da orientação do dispositivo. Deste modo, num cenário de lançamento do dispositivo numa capsula real, o modulo de comunicação deve ser colocado a uma distância razoável relativamente aos componentes de navegação, idealmente em faces ou mesmo em vértices opostos.

## Outras Informações:

- Perfil Github (organização) do projeto: https://github.com/X-MUON-PROBE;
- Cosmic rays: particles from outer space - CERN: https://home.cern/science/physics/cosmic-rays-particles-outer-space/