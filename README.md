# learn-trace

Laboratório com testes de racing usando Python e Opentelemetry

Estrutura baseada na documentação disponível na url: [https://opentelemetry.io/docs/languages/python/getting-started/](https://opentelemetry.io/docs/languages/python/getting-started/)

O exemplo a seguir usa uma aplicação [Flask](https://flask.palletsprojects.com/en/3.0.x/) simples.

---

## V1: Utilize o opentelemtry para auto instrumentação:


Para começar, configure um ambiente virtual, a partir da raiz do repositório clonado:

```sh
python3 -m venv venv
source ./venv/bin/activate
```

Instale o Framework Flask:

```sh
pip install flask
```

O arquivo v1/app.py possui a base inicial de código:

```sh
from random import randint
from flask import Flask, request
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@app.route("/rolldice")
def roll_dice():
    player = request.args.get('player', default = None, type = str)
    result = str(roll())
    return result

def roll():
    return randint(1, 6)
```

Faça uma chamada para validar o retorno e execução do código:

```sh
flask run -p 8080
```

> Deverá ser possível acessar a app a partir do servidor onde foi exposta na página 8080 utilizando a path /rolldice

A instrumentação automática gerará dados de telemetria, usaremos o agente do instrumento opentelemetry:

Instale o pacote opentelemetry-distro, que contém a API OpenTelemetry, SDK e também as ferramentas opentelemetry-bootstrap e opentelemetry-instrument que você usará abaixo:

```sh
pip install opentelemetry-distro
```

Execute o opentelemetry-bootstrap para detectar as bibliotecas instaladas e instalar automaticamente os pacotes de instrumentação:

```sh
opentelemetry-bootstrap -a install
```

A partir desta etapa já será possível visualizar os dados de rastreio gerados pela auto instrumentação:

```sh
export OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true
opentelemetry-instrument \
    --traces_exporter console \
    --metrics_exporter console \
    --logs_exporter console \
    --service_name dice-server \
    flask run -p 8080
```

Acesse novamente aaplicação exposta na porta 8080 utilizando a path /rolldice

**Verifique os dados de tracing sendo exibidos na saída padrão da console**

---

## V2: Incorpore spans customizados a aplicação:


A versão da app disponível na pasta v2 foi modifica para incluir o código que inicia um tracer e o utiliza para criar um trace dependente do trace gerado automaticamente:

```sh
cp v2/app.py app.py
cat app.py
```

A aplicação possuirá a estrutura abaixo:

```sh
from random import randint
from flask import Flask

from opentelemetry import trace

# Acquire a tracer
tracer = trace.get_tracer("diceroller.tracer")

app = Flask(__name__)

@app.route("/rolldice")
def roll_dice():
    return str(roll())

def roll():
    # This creates a new span that's the child of the current one
    with tracer.start_as_current_span("roll") as rollspan:
        res = randint(1, 6)
        rollspan.set_attribute("roll.value", res)
        return res
```

Execute a app novamente e valide o retorno na console:

```sh
export OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true
opentelemetry-instrument \
    --traces_exporter console \
    --metrics_exporter console \
    --logs_exporter console \
    --service_name dice-server \
    flask run -p 8080
```

---

## V3: inclua o Jeager para importar e visualizar o tracing da aplicação:

```sh
docker run -d --rm -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
    -p 80:16686 \ 
    -p 4317:4317 \
    -p 4318:4318 \
    -p 9411:9411 \
    jaegertracing/all-in-one:latest
```

> Neste cenário executamos um Jeager utilizando um container all-in-one para testar o funcionamento do coletor otpl que será invocado a seguir.

O Jeafer atuará como coletor das métricas e dados de rtracing enviados pelo apm, quanto a aplicação, uma última versão será usada neste teste

```sh
cp v3/app.py

cat app.py
```

A métrica utilizada nesta versão será um counter para contar o número de lançamentos para cada valor possível do dado:

```sh
# These are the necessary import declarations
from opentelemetry import trace
from opentelemetry import metrics

from random import randint
from flask import Flask, request
import logging

# Acquire a tracer
tracer = trace.get_tracer("diceroller.tracer")
# Acquire a meter.
meter = metrics.get_meter("diceroller.meter")

# Now create a counter instrument to make measurements with
roll_counter = meter.create_counter(
    "dice.rolls",
    description="The number of rolls by roll value",
)

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@app.route("/rolldice")
def roll_dice():
    # This creates a new span that's the child of the current one
    with tracer.start_as_current_span("roll") as roll_span:
        player = request.args.get('player', default = None, type = str)
        result = str(roll())
        roll_span.set_attribute("roll.value", result)
        # This adds 1 to the counter for the given roll value
        roll_counter.add(1, {"roll.value": result})
        return str(roll())

def roll():
    return randint(1, 6)
```

Instale o [OTLP exporter](https://opentelemetry.io/docs/specs/otel/protocol/exporter/):

```sh
pip install opentelemetry-exporter-otlp
```

Modifique o comando para exportar intervalos e métricas via OTLP execute a applicação novamente (Desta vez as saídas não serão redirecionadas ao stdout):

```sh
export OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true
opentelemetry-instrument --logs_exporter --service_name dice-server otlp flask run -p 8080
```

Execute algumas chamadas na aplicação na porta 8080 usando a path /rolldice, em seguida valide o conteúdo da coleta de trace na interface do Jeager na porta 80 do servidor ou host usado no laboratório.

---

##### Fiap - MBA
profhelder.pereira@fiap.com.br

**Lets Rock the Future**