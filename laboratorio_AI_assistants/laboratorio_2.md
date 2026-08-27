# Laboratorio: Vibe Coding, Pruebas Unitarias y Mocking en Python

**Duración estimada:** 30–45 min  
**Nivel:** Intermedio  
**Objetivo:** Realizar pruebas unitarias a través de asistente IA Copilot

---
## Prerrequisito:
- Realizar un fork de: https://github.com/wils0n/lab-github-copilot-unit-test/tree/main
- Copiar los archivos Calculadora.py, math_utils.py, weather_service.py de a este repositorio desde https://github.com/wils0n/laboratorios-md/tree/main/laboratorio_AI_assistants/files


## Ejercicio 1: Configuración global de tests con #codebase /setupTests

Las anotaciones como `#codebase /setupTests` se utilizan para indicar archivos o bloques de código que preparan el entorno de pruebas, por ejemplo, Te preguntará si deseas utilizar pytest o unittest:

```python
Elegiremos pytest
```

---

## Ejercicio 2: Generar Pruebas unitarias de una calculadora con #codebase /test

Si abrimos el archivo Calculator.py y en el prompt usas `#codebase /test`, generará los unit test para el archivo activo. Por ejemplo, para una calculadora te generará algo como:

```python
# #codebase /test
import pytest
from calculator import add, subtract, multiply, divide, power, factorial

def test_add():
	assert add(2, 3) == 5

def test_subtract():
	assert subtract(5, 2) == 3

def test_multiply():
	assert multiply(2, 4) == 8

def test_divide():
	assert divide(10, 2) == 5
	with pytest.raises(ValueError):
		divide(5, 0)

def test_power():
	assert power(2, 3) == 8

def test_factorial():
	assert factorial(5) == 120
	assert factorial(0) == 1
	with pytest.raises(ValueError):
		factorial(-1)
```

---

## Ejercicio 3: Generando tests para funciones matemáticas en math_utils.py

Abrir el archivo `math_utils.py`:

```python
def generate_fibonacci(n):
	"""Generates a list of the first n Fibonacci numbers."""
	if n < 0:
		raise ValueError("Input must be a non-negative integer.")
	fib_sequence = []
	a, b = 0, 1
	for _ in range(n):
		fib_sequence.append(a)
		a, b = b, a + b
	return fib_sequence

def factorial(n):
	"""Calculates the factorial of a number."""
	if n < 0:
		raise ValueError("Factorial is not defined for negative numbers.")
	result = 1
	for i in range(1, n + 1):
		result *= i
	return result
```

Para generar pruebas unitarias para estas funciones, puedes usar el siguiente prompt:

> Genera casos de prueba unitarios para math_utils.py

Y el resultado esperado sería:

```python
import pytest
from math_utils import generate_fibonacci, factorial

def test_generate_fibonacci():
	assert generate_fibonacci(0) == []
	assert generate_fibonacci(1) == [0]
	assert generate_fibonacci(5) == [0, 1, 1, 2, 3]
	assert generate_fibonacci(7) == [0, 1, 1, 2, 3, 5, 8]
	with pytest.raises(ValueError):
		generate_fibonacci(-1)

def test_factorial():
	assert factorial(0) == 1
	assert factorial(1) == 1
	assert factorial(5) == 120
	with pytest.raises(ValueError):
		factorial(-3)
```

Puedes modificar o ampliar los tests para cubrir más casos según lo necesites.

---

## Ejercicio 3: Generando tests con datos dummy para servicios externos

Supón que tienes el siguiente archivo `weather_service.py`:

```python
import requests

def get_weather(location):
	"""Fetches weather information for a given location."""
	response = requests.get(f"https://api.weather.com/v3/weather/{location}")
	return response.json()
```

Para generar pruebas unitarias usando mock y datos dummy, puedes usar el siguiente prompt:

> Genera casos de prueba unitarios usando mock para dependencias externas usando datos dummy

Y el resultado esperado sería:

```python
import pytest
from unittest.mock import patch, Mock
from weather_service import get_weather

def test_get_weather_with_mock():
	dummy_data = {"temp": 25, "condition": "Sunny"}
	mock_response = Mock()
	mock_response.json.return_value = dummy_data
	with patch("weather_service.requests.get", return_value=mock_response) as mock_get:
		result = get_weather("lima")
		mock_get.assert_called_once_with("https://api.weather.com/v3/weather/lima")
		assert result == dummy_data
```

Puedes modificar el `dummy_data` para simular diferentes respuestas y agregar más pruebas para otros escenarios (por ejemplo, manejo de errores).

---


## Ejercicio 4: Ejecutando las pruebas y revisando cobertura

1. Instala `pytest` en tu entorno de Python 3:

	```bash
	pip install pytest
	```

2. (Opcional pero recomendado) Instala el plugin `pytest-cov` para ver la cobertura de código:

	```bash
	pip install pytest-cov
	```

3. Ejecuta las pruebas con:

	```bash
	pytest
	```
	ó
	```bash
	python3 -m pytest
	```

4. Para ver el reporte de cobertura de código, ejecuta:

	```bash
	python3 -m pytest --cov
	```

	Esto mostrará un resumen de qué porcentaje de tu código está cubierto por las pruebas.

5. Deberías ver una salida indicando que las pruebas pasaron correctamente y, si usaste `--cov`, un reporte de cobertura.


---

## Conclusión

Has aprendido a:
- Configurar y usar `pytest`.
- Simular dependencias externas con `unittest.mock`.
- Escribir pruebas unitarias efectivas para código que depende de servicios externos.
- Entender el propósito de las anotaciones `#codebase /test` y `#codebase /setupTests` en la organización y automatización de pruebas.

¡Buen trabajo!
