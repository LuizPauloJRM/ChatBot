## Solução para receber e enviar mensagens no WhatsApp

https://waha.devlike.pro/

Vamos utilizar o Waha, uma api para envio e recebimento de mensagens no WhatsApp. Waha não utiliza API oficial do WhatsApp, mas tem algumas vantagens por ser extremamente simples de se utilizar e também por ser self hosted.

## Instalação do Docker e docker-compose

Vamos precisar do Docker e docker-compose instalados e funcionando na nossa máquina para começar a montar nosso projeto e subir nossos containers com os serviços.

### Windows

https://docs.docker.com/desktop/install/windows-install/

- Requisitos
    - Windows 10 ou superior atualizado
    - WSL versão 1.1.3 ou superior

### WSL

https://learn.microsoft.com/pt-br/windows/wsl/install

1. No PowerShell como administrador.

```powershell
wsl --install
```

1. Reiniciar e depois abrir a nova aplicação disponível “Ubuntu”. (Ubuntu é padrão)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/17354937-4524-429d-83e0-cf764a3a7c06/76016d00-32cb-449f-bbc5-e47769fb3885/Untitled.png)

1. Vai pedir para criar um usuário e senha para o sistema Linux. Após definir, está pronto para uso.

### Docker Desktop

1. Baixar instalador e instalar

https://docs.docker.com/desktop/install/windows-install/

1. Após reiniciar, já pode testar
### Docker Desktop

1. Baixar instalador e instalar

https://docs.docker.com/desktop/install/windows-install/

1. Após reiniciar, já pode testar

```powershell
docker run hello-world
```
## Iniciando o projeto

Agora vamos criar o diretório para o nosso projeto e também criar o Dockerfile e docker-compose.yml com as definições de nossos serviços.

### Waha

*docker-compose.yml*

```yaml
services:

  waha:
    image: devlikeapro/waha:latest
    container_name: wpp_bot_waha
    restart: always

    ports:
      - '3000:3000'

```

Para buildar e subir o serviço:

```bash
docker-compose up --build waha
```

Agora já podemos acessar o swagger e o dashboard do Waha em http://localhost:3000

### Api + Bot IA

Vamos utilizar Flask para construir nossa API/Webhook que vai receber e processar os eventos de mensagens.

*Instalar o Flask*

```bash
pip install flask
```

Não podemos esquecer de gerar o arquivo requirements.txt com todas as dependências do projeto:

*Para gerar o arquivo*

```bash
pip freeze > requirements.txt
```

Criar o arquivo principal da nossa aplicação que vai receber e processar os webhooks de eventos:

*app.py*

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/chatbot/webhook/', methods=['POST'])
def webhook():
    data = request.json

    print(f'EVENTO RECEBIDO: {data}')

    return jsonify({'status': 'success'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)

```

Agora para rodar e subir nosso serviço de API, vamos criar o Dockerfile e adicionar o serviço no nosso docker-compose:

*Dockerfile.api*

```docker
FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt requirements.txt
RUN python -m pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV FLASK_APP=app.py

EXPOSE 5000

CMD ["flask", "run", "--host=0.0.0.0", "--port=5000", "--debug"]

```

*.dockerignore*

```docker
venv

```

*docker-compose.yml*

```yaml
services:

  waha:
    image: devlikeapro/waha:latest
    container_name: wpp_bot_waha
    restart: always

    ports:
      - '3000:3000'

  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    container_name: wpp_bot_api
    ports:
      - '5000:5000'

```

Para buildar e subir o serviço:

```bash
docker-compose up --build api
```

Agora já temos um webhook disponível em http://localhost:5000/chatbot/webhook/ e podemos configurá-lo no dashboard do Waha para enviar eventos de mensagens para o endpoint processar.

**Importante:** Na configuração do Waha, a url do webhook deve usar o nome do serviço que declaramos no docker-compose.yml (api): http://api:5000/chatbot/webhook/

O único evento que queremos gerar eventos e enviar para o webhook é o evento **message**.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/17354937-4524-429d-83e0-cf764a3a7c06/41d3c416-ca82-4af5-9d94-fb775ba71aad/image.png)

### Responder mensagens

Agora que já estamos recebendo na nossa API as mensagens recebidas no WhatsApp, vamos criar um client da Api do Waha no projeto, e implementar um método para responder mensagens recebidas.

*Primeiro precisamos instalar a biblioteca requests e refazer o requirements.txt:*

```bash
pip install requests
pip freeze > requirements.txt
```

Agora criar o diretório **services** na raiz do projeto ****com os arquivos:

***services/__init__**.py*

*services/waha.py*

```python
import requests

class Waha:

    def __init__(self):
        self.__api_url = 'http://waha:3000'

    def send_message(self, chat_id, message):
        url = f'{self.__api_url}/api/sendText'
        headers = {
            'Content-Type': 'application/json',
        }
        payload = {
            'session': 'default',
            'chatId': chat_id,
            'text': message,
        }
        requests.post(
            url=url,
            json=payload,
            headers=headers,
        )

```

Agora vamos usar nosso client do Waha para responder as mensagens recebidas:

*app.py*

```python
from flask import Flask, request, jsonify

from services.waha import Waha

app = Flask(__name__)

@app.route('/chatbot/webhook/', methods=['POST'])
def webhook():
    data = request.json

    print(f'EVENTO RECEBIDO: {data}')

    waha = Waha()

    chat_id = data['payload']['from']
    waha.send_message(
        chat_id=chat_id,
        message='Resposta Automática :)',
    )

    return jsonify({'status': 'success'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)

```

Agora podemos buildar novamente o serviço e testar:

```bash
docker-compose up --build api
```

Se quiser enfeitar e adicionar um “*Digitando…*” :)

*services/waha.py*

```python
import requests

class Waha:

    def __init__(self):
        self.__api_url = 'http://waha:3000'

    def send_message(self, chat_id, message):
        url = f'{self.__api_url}/api/sendText'
        headers = {
            'Content-Type': 'application/json',
        }
        payload = {
            'session': 'default',
            'chatId': chat_id,
            'text': message,
        }
        requests.post(
            url=url,
            json=payload,
            headers=headers,
        )

    def start_typing(self, chat_id):
        url = f'{self.__api_url}/api/startTyping'
        headers = {
            'Content-Type': 'application/json',
        }
        payload = {
            'session': 'default',
            'chatId': chat_id,
        }
        requests.post(
            url=url,
            json=payload,
            headers=headers,
        )

    def stop_typing(self, chat_id):
        url = f'{self.__api_url}/api/stopTyping'
        headers = {
            'Content-Type': 'application/json',
        }
        payload = {
            'session': 'default',
            'chatId': chat_id,
        }
        requests.post(
            url=url,
            json=payload,
            headers=headers,
        )

```

*app.py*

```python
import time
import random

from flask import Flask, request, jsonify

from services.waha import Waha

app = Flask(__name__)

@app.route('/chatbot/webhook/', methods=['POST'])
def webhook():
    data = request.json

    print(f'EVENTO RECEBIDO: {data}')

    waha = Waha()

    chat_id = data['payload']['from']

    waha.start_typing(chat_id=chat_id)
    time.sleep(random.randint(3, 10))
    waha.send_message(
        chat_id=chat_id,
        message='Resposta Automática :)',
    )
    waha.stop_typing(chat_id=chat_id)

    return jsonify({'status': 'success'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)

```

### Adicionando IA

Vamos utilizar o modelo/llm open source llama3.1 da Meta, rodando através da Groq.

https://groq.com/

Primeiro vamos criar um arquivo de variáveis de ambiente no nosso projeto para armazenar a API KEY da Groq gerada no painel:

*.env*

```
GROQ_API_KEY='YOUR GROQ API KEY'
```

Agora vamos instalar algumas dependências que vamos usar:

```bash
pip install langchain
pip install langchain_groq
pip install python-decouple

```

Vamos atualizar o requirements.txt:

```bash
pip freeze > requirements.txt
```

Já podemos criar nosso bot de IA que vai gerar as respostas das mensagens. Para isso, vamos criar um diretório no nosso projeto chamado **bot** e criar nele os seguintes arquivos:

***bot/__init__**.py*

*bot/ai_bot.py*

```python
import os

from decouple import config

from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import PromptTemplate
from langchain_groq import ChatGroq

os.environ['GROQ_API_KEY'] = config('GROQ_API_KEY')

class AIBot:

    def __init__(self):
        self.__chat = ChatGroq(model='llama-3.1-70b-versatile')

    def invoke(self, question):
        prompt = PromptTemplate(
            input_variables=['texto'],
            template='''
            Você é um tradutor de textos que traduz o texto do usuário para Italiano.
            <texto>
            {texto}
            </texto>
            '''
        )
        chain = prompt | self.__chat | StrOutputParser()
        response = chain.invoke({
            'texto': question,
        })
        return response

```

Com isso, já podemos usar nosso bot para responder as mensagens recebidas:

*app.py*

```python
from flask import Flask, request, jsonify

from bot.ai_bot import AIBot
from services.waha import Waha

app = Flask(__name__)

@app.route('/chatbot/webhook/', methods=['POST'])
def webhook():
    data = request.json

    print(f'EVENTO RECEBIDO: {data}')

    waha = Waha()
    ai_bot = AIBot()

    chat_id = data['payload']['from']
    received_message = data['payload']['body']

    waha.start_typing(chat_id=chat_id)

    response = ai_bot.invoke(question=received_message)
    waha.send_message(
        chat_id=chat_id,
        message=response,
    )
    waha.stop_typing(chat_id=chat_id)

    return jsonify({'status': 'success'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)

```

Vamos buildar e subir o serviço atualizado para testar: