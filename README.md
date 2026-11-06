# Alarme de Temperatura com ESP32 (MicroPython)

Monitora a temperatura com um sensor **DHT11** e, quando ela chega ao limite configurado
(padrão **23 °C**), o **LED pisca** e o **buzzer apita** sem parar até a temperatura cair
ou o usuário encerrar o programa digitando `q`.

Escrito em **MicroPython** e executado pela IDE **Thonny**.

---

## Componentes

| Componente | Modelo usado | Observação |
|---|---|---|
| Placa | ESP32 (ESP32-D0WD-V3, DevKit) | MicroPython v1.29.0 |
| Sensor de temperatura e umidade | DHT11 (azul) | Faixa 0–50 °C, ±2 °C |
| LED | LED azul da própria placa (GPIO2) | Nenhuma ligação extra |
| Buzzer | PUI Audio AT-1224-TWT-5V-2-R | Transdutor magnético **passivo** (precisa de PWM), ~2 kHz |
| Resistor | 100 Ω (marrom-preto-marrom) | Limita a corrente do pino do buzzer |
| Diodo (opcional) | 1N4148 ou 1N4007 | Em paralelo com o buzzer, faixa no (+) |

---

## Ligações

| Componente | Pino do componente | Pino do ESP32 |
|---|---|---|
| DHT11 | VCC (+) | 3V3 |
| DHT11 | DATA (OUT/S) | **GPIO4** (D4) |
| DHT11 | GND (−) | GND |
| Buzzer | (+) | **GPIO18** (D18) através do resistor de 100 Ω |
| Buzzer | (−) | GND |
| LED | — | GPIO2 (LED interno da placa) |

```
GPIO4  ───────────────────► DATA  (DHT11)
3V3    ───────────────────► VCC   (DHT11)
GND    ───────────────────► GND   (DHT11)

GPIO18 ──[ 100 Ω ]────────► (+) buzzer
GND    ───────────────────► (−) buzzer
```

> **Não use GPIO0 nem GPIO1 no ESP32.** O GPIO1 é o TX da USB (é por ele que o Thonny
> conversa com a placa) e o GPIO0 é o pino de boot.

---

## Instalação do MicroPython na placa

1. Instale o [Thonny](https://thonny.org).
2. Conecte o ESP32 via USB. Se não aparecer porta COM, instale o driver USB
   (CP210x ou CH340, conforme o chip ao lado do conector).
3. No Thonny: **Ferramentas > Opções > Interpretador**
   - Interpretador: **MicroPython (ESP32)**
   - Porta: a COM da placa (ex.: COM6)
4. Clique em **Install or update MicroPython (esptool)**:
   - MicroPython family: **ESP32**
   - Variant: **Espressif • ESP32 / WROOM**
   - Marque **Erase all flash before installing**
   - Clique em **Install**
5. Se der erro de conexão: segure o botão **BOOT**, aperte e solte **EN**, clique em
   **Install** e solte o BOOT quando começar a gravação.
6. Clique em **Stop/Restart**. O Shell deve mostrar `MicroPython v1.29.0 ...` e `>>>`.

---

## Arquivos

| Arquivo | O que faz |
|---|---|
| `alarme_temperatura_led_buzzer_esp32.py` | **Programa principal**: LED pisca + buzzer apita a partir do limite |
| `monitor_temperatura_alerta_led_esp32.py` | Versão só com LED fazendo papel de alarme |
| `monitor_temperatura_esp32.py` | Monitor com LED aceso acima de 25 °C (sem piscar) |
| `menu_temperatura_led_esp32.py` | Menu de teste: `1` lê o sensor, `2` testa o LED, `0` sai |

---

## Como usar

1. Abra `alarme_temperatura_led_buzzer_esp32.py` no Thonny.
2. Aperte **F5** (Executar).
3. Um **bip curto** com o LED aceso confirma que o buzzer está ligado corretamente.
4. A cada 2 s o Shell mostra a leitura:

   ```
   Atual: 23.6 C | Media: 23.6 C | Umidade: 73 % | Alarme: LIGADO
   ```

5. Para parar: digite **`q`** e aperte **Enter** (ou **Ctrl+C**). LED e buzzer são desligados.

### Rodar automaticamente ao ligar a placa

Salve o arquivo na placa com o nome **`main.py`**:
**Arquivo > Salvar como > MicroPython device > `main.py`**.

Para remover: **Exibir > Arquivos**, botão direito em `main.py` na placa > **Excluir**.

---

## Configurações

Todas ficam no início do programa principal:

| Variável | Padrão | Função |
|---|---|---|
| `LIMITE` | `23.0` | Temperatura (°C) que dispara o alarme |
| `HISTERESE` | `0.5` | O alarme só desliga abaixo de `LIMITE - HISTERESE` (evita liga/desliga) |
| `MEDIA_N` | `3` | Leituras na média móvel (use `1` para reagir na hora) |
| `INTERVALO_MS` | `2000` | Intervalo entre leituras do sensor (DHT11: mínimo ~1 s) |
| `VELOCIDADE_MS` | `250` | Velocidade da piscada e do bip (menor = mais rápido) |
| `FREQ_BUZZER` | `2048` | Frequência do som (ajuste para a mais alta do seu buzzer) |

---

## Como o código funciona

- **Leitura do DHT11 com casa decimal:** o `dht.DHT11` do MicroPython descarta o decimal.
  O código lê os bytes brutos (`sensor.buf`) e monta `inteiro + decimal / 10`.
- **Tentativas e filtro:** até 3 tentativas por leitura; valores fora de 0–50 °C ou
  5–95 % são descartados.
- **Média móvel:** suaviza oscilações antes de decidir o alarme.
- **Timer:** um `machine.Timer` chama a função `piscar()` a cada `VELOCIDADE_MS`. Ela
  inverte o LED e liga/desliga o buzzer junto. Assim a piscada não para enquanto o
  código espera o sensor.
- **Leitura do teclado sem travar:** `select.poll()` verifica se foi digitado `q` sem
  bloquear o loop.
- **Encerramento seguro:** o bloco `finally` para o Timer e desliga LED e buzzer.

---

## Solução de problemas

| Sintoma | Causa | Solução |
|---|---|---|
| "Device is busy or does not respond" e números sem parar | Placa com programa **Arduino** gravado (não é MicroPython) | Instalar o MicroPython (ver seção acima) |
| `IndentationError: unexpected indent` | Espaço sobrando no começo de alguma linha ao colar | `Ctrl+A`, apagar e colar de novo, ou abrir o arquivo `.py` |
| "Erro ao ler o sensor" | Ligação errada ou tipo de sensor errado | Conferir DATA no GPIO4, VCC no 3V3; testar `DHT11` e `DHT22` |
| Temperatura absurda (ex.: 589 °C, umidade 1869 %) | Sensor DHT11 lido como DHT22 | Usar `dht.DHT11(Pin(4))` |
| Sensor só muda de 1 em 1 grau | Limitação do DHT11 | Normal; para mais precisão use DHT22 ou DS18B20 |
| LED funciona, buzzer mudo | Pino, polaridade ou resistor errados | Conferir D18, inverter pernas, resistor marrom-preto-marrom |
| Buzzer baixo | Buzzer de 5 V alimentado com 3,3 V | Ver "Aumentar o volume" abaixo |

### Teste rápido do buzzer (no Shell)

```python
from machine import Pin, PWM; import time
b = PWM(Pin(18), freq=2048, duty=512); time.sleep(2); b.duty(0)
```

### Aumentar o volume do buzzer

1. **Achar a frequência mais alta** e colocar em `FREQ_BUZZER`:

   ```python
   from machine import Pin, PWM; import time
   b = PWM(Pin(18), freq=2000, duty=0)
   for f in (1500, 2000, 2048, 2300, 2400, 2700, 3000, 3500, 4000):
       print(f); b.freq(f); b.duty(512); time.sleep(0.8); b.duty(0); time.sleep(0.4)
   ```

2. **Resistor menor:** 68 Ω (mínimo 47 Ω sem transistor).
3. **Dois pinos em oposição:** (−) do buzzer no **GPIO19** em vez do GND, com PWM
   invertido. Dobra a variação de tensão no buzzer.

   ```python
   a = PWM(Pin(18), freq=2048, duty=512)
   b = PWM(Pin(19), freq=2048, duty=512, invert=True)
   ```

4. **Transistor (BC547 / 2N2222) + 5 V (pino VIN):** volume máximo do buzzer.

---

## Próximos passos

- [ ] Definir a frequência de maior volume do buzzer
- [ ] Aumentar o volume (dois pinos ou transistor)
- [ ] Gravar como `main.py` para funcionar sem o computador
- [ ] (Opcional) Trocar o DHT11 por DHT22 ou DS18B20 para mais precisão
