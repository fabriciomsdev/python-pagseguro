> **Este projeto é um fork do projeto original que é mantido pelo @PagSeguro (@Japple). \n Este projeto apenas contém implementaçõaes extras como por exemplo a criação de planos de recorrência e outros**. 

Integração com a API v2 de pagamentos e notificações do Pagseguro utilizando requests.

Instalando
==========================
```bash
pip install pagseguro
```

ou


```bash
pip install -e git+https://github.com/fabriciomsdev/python-pagseguro#egg=pagseguro
```

ou

```
git clone https://github.com/fabriciomsdev/python-pagseguro
cd python-pagseguro
pip install -r requirements.txt
python setup.py install
```

Rodando os testes
=================

```
make test
```

Como usar
=========

### Carrinho de compras / ordem de venda

Uma instancia de **PagSeguro** funciona como uma ordem de venda, ou um carrinho de compras. É preciso criar a instancia passando como parametro email e token.

> Opcionalmente é possivel passar o parametro **data** contendo valores a serem passados diretamente para a API.

```python
from pagseguro import PagSeguro

pg = PagSeguro(email="seuemail@dominio.com", token="ABCDEFGHIJKLMNO")
```

### Sandbox e Config Customizadas

Ao instanciar um objecto `PagSeguro`, você poderá passar um parâmetro `config` contendo a class de configuração a ser usada pela classe. A variável `config` somente irá aceitar o tipo `dict`.

```python
from pagseguro import PagSeguro

config = {'sandbox': True}
pg = PagSeguro(email="seuemail@dominio.com", token="ABCDEFGHIJKLMNO", config=config)
```

O seu config também pode fazer override de algumas váriaveis pré-definidas na classe de Config padrão. São elas:

- CURRENCY - Moeda utilizada. Valor padrão: `'BRL'`
- DATETIME_FORMAT - Formato de Data/Hora. Valor Padrão: `'%Y-%m-%dT%H:%M:%S'`
- REFERENCE_PREFIX - Formato do valor de referência do produto. Valor Padrão: `'REF%s'` Obs: Nesse caso, sempre é necessário deixar o `%s` ao final do prefixo para que o mesmo seja preenchido automaticamente
- USE_SHIPPING - User endereço de entrega. Valor padrão: `True`


### Configurando os dados do comprador

```python
pg.sender = {
    "name": "Bruno Rocha",
    "area_code": 11,
    "phone": 981001213,
    "email": "rochacbruno@gmail.com",
}
```

### Configurando endereço de entrega
```python
pg.shipping = {
    "type": pg.SEDEX,
    "street": "Av Brig Faria Lima",
    "number": 1234,
    "complement": "5 andar",
    "district": "Jardim Paulistano",
    "postal_code": "06650030",
    "city": "Sao Paulo",
    "state": "SP",
    "country": "BRA"
}
```

Caso o **country** não seja informado o valor default será "BRA"

O **type** pode ser pg.SEDEX, pg.PAC, pg.NONE
> Opcionalmente pode ser numerico seguindo a tabela pagseguro:

| Número | Descrição | Type |
| ------ | --------- | ---- |
| 1 | PAC | pg.PAC |
| 2 | SEDEX | pg.SEDEX |
| 3 | Nao especificado | pg.NONE |

Valores opcionais para **shipping**
- "cost": "123456.26"
    Decimal, com duas casas decimais separadas por ponto (p.e, 1234.56), maior que 0.00 e menor ou igual a 9999999.00.


### Configurando a referencia

Referencia é geralmente o código que identifica a compra em seu sistema

Por padrão a referencia será prefizada com "REF", mas isso pode ser alterado setando um prefixo diferente

```python
pg.reference_prefix = "CODE"
pg.reference_prefix = None  # para desabilitar o prefixo
```

```python
pg.reference = "00123456789"
print pg.reference
"REF00123456789"
```

### Configurando valor extra

Especifica um valor extra que deve ser adicionado ou subtraído ao valor total do pagamento. Esse valor pode representar uma taxa extra a ser cobrada no pagamento ou um desconto a ser concedido, caso o valor seja negativo.

Formato: Float (positivo ou negativo).

```python
pg.extra_amount = 12.70
```

### Inserindo produtos no carrinho

O carrinho de compras é uma lista contendo dicionários que representam cada produto nos seguinte formato.

#### adicionando vários produtos

```python
pg.items = [
    {"id": "0001", "description": "Produto 1", "amount": 354.20, "quantity": 2, "weight": 200},
    {"id": "0002", "description": "Produto 2", "amount": 50, "quantity": 1, "weight": 1000}
]
```

O **weight** do produto é representado em gramas

#### Adicionando um produto por vez

Da forma tradicional

```python
pg.items.append(
    {"id": "0003", "description": "Produto 3", "amount": 354.20, "quantity": 2, "weight": 200},
)
```

Ou atraves do helper

```python
pg.add_item(id="0003", description="produto 4", amount=320, quantity=1, weight=2500)
```

### Configurando a URL de redirect

Para onde o comprador será redirecionado após completar o pagamento

```python
pg.redirect_url = "http://meusite.com/obrigado"
```

### Configurando a URL de notificaçao (opcional)

```python
pg.notification_url = "http://meusite.com/notification"
```

### Efetuando o processo de checkout

Depois que o carrinho esta todo configurado e com seus itens inseridos, ex: quando o seu cliente clicar no botão "efetuar pagamento", o seguinte método deverá ser executado.

```python
response = pg.checkout()
```

O método checkout faz a requisição ao pagseguro e retorna um objeto PagSeguroResponse com os atributos code, date, payment_url, errors.

É aconselhavel armazenar o código da transação em seu banco de dados juntamente com as informações do carrinho para seu controle interno.

Utilize a **payment_url** para enviar o comprador para a página de pagamento do pagseguro.

```python
return redirect(response.payment_url)
```

Após o pagamento o comprador será redirecionado de volta para os eu site através da configuração de url de retorno global ou utilizará a url especificada no parametro **redirect_url**

# Notificações

O PagSeguro envia as notificações para a URL que você configurou usando o protocolo HTTP, pelo método POST.

Suponde que você receberá a notificação em: http://seusite.com/notification

> Pseudo codigo

```python
from pagseguro import PagSeguro

def notification_view(request):
    notification_code = request.POST['notificationCode']
    pg = PagSeguro(email="seuemail@dominio.com", token="ABCDEFGHIJKLMNO")
    notification_data = pg.check_notification(notification_code)
    ...
```

No exemplo acima pegamos o **notificationCode** que foi enviado através do pagseguro e fizemos uma consulta para pegar os dados da notificação, o retorno será em um dicionário Python com o seguinte formato:

```python
{
    "date": datetime(2013, 01, 01, 18, 23, 0000),
    "code": "XDFD454545",
    "reference": "REF00123456789",
    "type": 1,
    "status": 3,
    "cancellationSource": "INTERNAL",
    ...
}
```

A lista completa de valores pode ser conferida em  https://pagseguro.uol.com.br/v2/guia-de-integracao/api-de-notificacoes.html


# Implementações

> Implementações a serem feitas, esperando o seu Pull Request!!!

## Quokka CMS
[Quokka Cart PagSeguro Processor](https://github.com/pythonhub/quokka-cart/blob/master/processors/pagseguro_processor.py)

## Exemplo em Flask

[FlaskSeguro](https://github.com/rochacbruno/python-pagseguro/tree/master/examples/flask)
by @shyba




