# Desafio-DIO-Criando-Uma-API-Com-FastAPI-Utilizando-TDD
Passo 1: Configuração do Ambiente e Dependências
Crie uma estrutura de diretórios:
project/
├── app/
│   ├── __init__.py
│   ├── controllers/
│   │   ├── __init__.py
│   │   └── product_controller.py
│   ├── exceptions.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── product.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   └── product_schema.py
│   └── usecases/
│       ├── __init__.py
│       └── product_usecase.py
├── tests/
│   ├── __init__.py
│   ├── test_product_controller.py
│   ├── test_product_schema.py
│   └── test_product_usecase.py
├── requirements.txt
└── run.py
app/: Contém o código principal da aplicação.
tests/: Diretório para os testes.
requirements.txt: Lista das dependências do Python.
Instale as dependências necessárias:

Crie um arquivo requirements.txt com as dependências necessárias:
pytest
marshmallow
Instale as dependências com:
pip install -r requirements.txt
Passo 2: Implementação dos Componentes da Aplicação
Models
Crie o modelo Product em app/models/product.py:

from datetime import datetime
from sqlalchemy import Column, Integer, String, Float, DateTime
from app import db

class Product(db.Model):
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    price = Column(Float, nullable=False)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    def __repr__(self):
        return f'<Product {self.name}>'
Schemas
Crie o schema ProductSchema em app/schemas/product_schema.py:
from marshmallow import Schema, fields, validate

class ProductSchema(Schema):
    id = fields.Int(dump_only=True)
    name = fields.Str(required=True)
    price = fields.Float(required=True, validate=validate.Range(min=0))
    updated_at = fields.DateTime(dump_only=True)
Controllers
Crie a controller ProductController em app/controllers/product_controller.py:
from app import db
from app.models.product import Product
from app.schemas.product_schema import ProductSchema
from app.exceptions import NotFoundError

product_schema = ProductSchema()
products_schema = ProductSchema(many=True)

class ProductController:
    @staticmethod
    def create_product(data):
        new_product = Product(
            name=data['name'],
            price=data['price']
        )
        db.session.add(new_product)
        db.session.commit()
        return product_schema.dump(new_product)

    @staticmethod
    def update_product(product_id, data):
        product = Product.query.get(product_id)
        if not product:
            raise NotFoundError('Product not found.')

        product.name = data.get('name', product.name)
        product.price = data.get('price', product.price)
        product.updated_at = datetime.utcnow()  # Update updated_at to current time
        db.session.commit()
        return product_schema.dump(product)

    @staticmethod
    def get_products_by_price_range(min_price, max_price):
        products = Product.query.filter(Product.price > min_price, Product.price < max_price).all()
        return products_schema.dump(products)
Use Cases
Crie o use case ProductUseCase em app/usecases/product_usecase.py:
from app.controllers.product_controller import ProductController

class ProductUseCase:
    @staticmethod
    def create_product(data):
        return ProductController.create_product(data)

    @staticmethod
    def update_product(product_id, data):
        return ProductController.update_product(product_id, data)

    @staticmethod
    def get_products_by_price_range(min_price, max_price):
        return ProductController.get_products_by_price_range(min_price, max_price)
Exceptions
Crie um arquivo app/exceptions.py para gerenciar exceções customizadas:
class NotFoundError(Exception):
    def __init__(self, message):
        super().__init__(message)
        self.message = message
Passo 3: Implementação dos Testes
Testes de Schemas
Crie o arquivo de teste tests/test_product_schema.py:
import pytest
from app.schemas.product_schema import ProductSchema

@pytest.fixture
def product_schema():
    return ProductSchema()

def test_product_schema_validate_price_valid(product_schema):
    valid_data = {'name': 'Test Product', 'price': 100.0}
    errors = product_schema.validate(valid_data)
    assert not errors

def test_product_schema_validate_price_negative(product_schema):
    invalid_data = {'name': 'Test Product', 'price': -10.0}
    errors = product_schema.validate(invalid_data)
    assert 'price' in errors
Testes de Use Cases
Crie o arquivo de teste tests/test_product_usecase.py:
import pytest
from unittest.mock import patch
from datetime import datetime
from app.usecases.product_usecase import ProductUseCase
from app.models.product import Product
from app.exceptions import NotFoundError

@pytest.fixture
def mock_product():
    return Product(id=1, name='Test Product', price=100.0)

@patch('app.controllers.product_controller.ProductController.create_product', return_value={'id': 1, 'name': 'Test Product', 'price': 100.0})
def test_create_product(mock_create_product):
    data = {'name': 'Test Product', 'price': 100.0}
    result = ProductUseCase.create_product(data)
    assert result['name'] == 'Test Product'

@patch('app.controllers.product_controller.ProductController.update_product', return_value={'id': 1, 'name': 'Updated Product', 'price': 150.0, 'updated_at': datetime.utcnow()})
def test_update_product(mock_update_product):
    product_id = 1
    data = {'name': 'Updated Product', 'price': 150.0}
    result = ProductUseCase.update_product(product_id, data)
    assert result['name'] == 'Updated Product'

@patch('app.controllers.product_controller.ProductController.get_products_by_price_range', return_value=[{'id': 1, 'name': 'Test Product', 'price': 6000.0, 'updated_at': datetime.utcnow()}])
def test_get_products_by_price_range(mock_get_products):
    min_price = 5000.0
    max_price = 8000.0
    results = ProductUseCase.get_products_by_price_range(min_price, max_price)
    assert len(results) == 1
    assert results[0]['price'] > 5000.0 and results[0]['price'] < 8000.0
Testes de Controllers (Teste de Integração)
Crie o arquivo de teste tests/test_product_controller.py:
import pytest
from unittest.mock import patch
from datetime import datetime
from app.controllers.product_controller import ProductController
from app.models.product import Product
from app.exceptions import NotFoundError

@pytest.fixture
def mock_product():
    return Product(id=1, name='Test Product', price=100.0)

def test_create_product(mock_product):
    data = {'name': 'Test Product', 'price': 100.0}
    result = ProductController.create_product(data)
    assert result['name'] == 'Test Product'

def test_update_product(mock_product):
    product_id = 1
    data = {'name': 'Updated Product', 'price': 150.0}
    result = ProductController.update_product(product_id, data)
    assert result['name'] == 'Updated Product'

def test_update_product_not_found(mock_product):
    with pytest.raises(NotFoundError):
        ProductController.update_product(999, {'name': 'Updated Product', 'price': 150.0})

def test_get_products_by_price_range(mock_product):
    min_price = 5000.0
    max_price = 8000.0
    results = ProductController.get_products_by_price_range(min_price, max_price)
    assert not results  # Assuming no products in this price range for the mock data
Passo 4: Executando os Testes
Para executar os testes com pytest, basta rodar o seguinte comando na raiz do projeto:
pytest
Isso vai executar todos os testes dentro do diretório tests/ e verificar se tudo está funcionando corretamente conforme as especificações.
Este exemplo cobre os requisitos básicos mencionados, incluindo a criação de produtos, atualização com tratamento de exceções e filtro por faixa de preço. 
