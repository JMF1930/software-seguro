# Solución Lab Presupuesto

Tomo los datos de la respuesta del GET luego del logueo:

```json
[
  {
    "id": 1,
    "titulo": "Flete mercadería",
    "categoria": {
      "id": 2,
      "clave": "transporte",
      "nombre": "Transporte"
    },
    "monto": "6000.00",
    "fecha": "2023-09-04",
    "revisado": false
  },
  {
    "id": 2,
    "titulo": "Alquiler local",
    "categoria": {
      "id": 4,
      "clave": "esenciales",
      "nombre": "Esenciales"
    },
    "monto": "10000.00",
    "fecha": "2023-08-15",
    "revisado": false
  },
  {
    "id": 3,
    "titulo": "Luz",
    "categoria": {
      "id": 4,
      "clave": "esenciales",
      "nombre": "Esenciales"
    },
    "monto": "2000.00",
    "fecha": "2023-08-22",
    "revisado": true
  },
  {
    "id": 4,
    "titulo": "Agua",
    "categoria": {
      "id": 5,
      "clave": "alimentos",
      "nombre": "Alimentos"
    },
    "monto": "1000.00",
    "fecha": "2023-09-01",
    "revisado": false
  },
  {
    "id": 5,
    "titulo": "Internet",
    "categoria": {
      "id": 4,
      "clave": "esenciales",
      "nombre": "Esenciales"
    },
    "monto": "500.00",
    "fecha": "2023-09-15",
    "revisado": true
  },
  {
    "id": 6,
    "titulo": "Nómina empleados",
    "categoria": {
      "id": 4,
      "clave": "esenciales",
      "nombre": "Esenciales"
    },
    "monto": "50000.00",
    "fecha": "2023-08-01",
    "revisado": false
  },
  {
    "id": 7,
    "titulo": "Marketing",
    "categoria": {
      "id": 3,
      "clave": "varios",
      "nombre": "Varios"
    },
    "monto": "10000.00",
    "fecha": "2023-09-05",
    "revisado": false
  },
  {
    "id": 8,
    "titulo": "Publicidad",
    "categoria": {
      "id": 3,
      "clave": "varios",
      "nombre": "Varios"
    },
    "monto": "5000.00",
    "fecha": "2023-09-22",
    "revisado": false
  },
  {
    "id": 9,
    "titulo": "Reuniones",
    "categoria": {
      "id": 3,
      "clave": "varios",
      "nombre": "Varios"
    },
    "monto": "2000.00",
    "fecha": "2023-10-01",
    "revisado": false
  },
  {
    "id": 10,
    "titulo": "Reparación de equipo informático",
    "categoria": {
      "id": 3,
      "clave": "varios",
      "nombre": "Varios"
    },
    "monto": "10000.00",
    "fecha": "2023-08-10",
    "revisado": false
  },
  {
    "id": 11,
    "titulo": "Seguros",
    "categoria": {
      "id": 6,
      "clave": "impuestos",
      "nombre": "Impuestos"
    },
    "monto": "5000.00",
    "fecha": "2023-08-20",
    "revisado": false
  },
  {
    "id": 12,
    "titulo": "Licencias de software",
    "categoria": {
      "id": 3,
      "clave": "varios",
      "nombre": "Varios"
    },
    "monto": "2000.00",
    "fecha": "2023-09-01",
    "revisado": false
  },
  {
    "id": 13,
    "titulo": "Formación de empleados",
    "categoria": {
      "id": 3,
      "clave": "varios",
      "nombre": "Varios"
    },
    "monto": "5000.00",
    "fecha": "2023-10-01",
    "revisado": false
  }
]
```

Tabla original:

| **Título**                       | **Categoría** | **Monto** | **Fecha** | **Revisado** |
|----------------------------------|---------------|-----------|-----------|--------------|
| Flete mercadería                 | Transporte    | 6000,00   | 4/9/2023  | No           |
| Alquiler local                   | Esenciales    | 10000,00  | 15/8/2023 | No           |
| Luz                              | Esenciales    | 2000,00   | 22/8/2023 | No           |
| Agua                             | Alimentos     | 1000,00   | 1/9/2023  | No           |
| Internet                         | Esenciales    | 500,00    | 15/9/2023 | No           |
| Nómina empleados                 | Esenciales    | 50000,00  | 1/8/2023  | No           |
| Marketing                        | Varios        | 10000,00  | 5/9/2023  | No           |
| Publicidad                       | Varios        | 5000,00   | 22/9/2023 | No           |
| Reuniones                        | Varios        | 2000,00   | 1/10/2023 | No           |
| Reparación de equipo informático | Varios        | 10000,00  | 10/8/2023 | No           |
| Seguros                          | Impuestos     | 5000,00   | 20/8/2023 | No           |
| Licencias de software            | Varios        | 2000,00   | 1/9/2023  | No           |
| Formación de empleados           | Varios        | 5000,00   | 1/10/2023 | No           |

| Monto promedio:             |     | 8346,15  |
|-----------------------------|-----|----------|
| Monto máximo:               |     | 50000,00 |
| Monto mínimo:               |     | 500,00   |
| Promedio gastos decoración: |     | 0,00     |
| Promedio gastos transporte: |     | 6000,00  |
| Promedio gastos varios:     |     | 5666,67  |
| Promedio gastos esenciales: |     | 15625,00 |
| Promedio gastos impuestos:  |     | 5000,00  |
| Promedio gastos alimentos:  |     | 1000,00  |

Valores modificados:

| **Título**                       | **Categoría** | **Nuevo monto** | **Fecha** | **Revisado** |
|----------------------------------|---------------|-----------------|-----------|--------------|
| Flete mercadería                 | Transporte    | 1500,00         | 4/9/2023  | Si           |
| Alquiler local                   | Esenciales    | 13000,00        | 15/8/2023 | Si           |
| Luz                              | Esenciales    | 2000,00         | 22/8/2023 | Si           |
| Agua                             | Alimentos     | 500,00          | 1/9/2023  | Si           |
| Internet                         | Esenciales    | 500,00          | 15/9/2023 | Si           |
| Nómina empleados                 | Esenciales    | 50000,00        | 1/8/2023  | Si           |
| Marketing                        | Varios        | 10000,00        | 5/9/2023  | Si           |
| Publicidad                       | Varios        | 5000,00         | 22/9/2023 | Si           |
| Reuniones                        | Varios        | 2000,00         | 1/10/2023 | Si           |
| Reparación de equipo informático | Varios        | 12000,00        | 10/8/2023 | Si           |
| Seguros                          | Impuestos     | 500,00          | 20/8/2023 | Si           |
| Licencias de software            | Varios        | 2000,00         | 1/9/2023  | Si           |
| Formación de empleados           | Varios        | 5000,00         | 1/10/2023 | Si           |
|                                  |               |                 |           |              |
| Monto promedio:                  |               | 8000,00         |           |              |
| Monto máximo:                    |               | 50000,00        |           |              |
| Monto mínimo:                    |               | 500,00          |           |              |
| Promedio gastos decoración:      |               | 0,00            |           |              |
| Promedio gastos transporte:      |               | 1500,00         |           |              |
| Promedio gastos varios:          |               | 6000,00         |           |              |
| Promedio gastos esenciales:      |               | 16375,00        |           |              |
| Promedio gastos impuestos:       |               | 5000,00         |           |              |
| Promedio gastos alimentos:       |               | 1500,00         |           |              |

POST modificando los datos:

Posicionamiento y luego modificación.

<https://chl-3b0862a5-bc83-4f06-9d7c-32ba62110b25-presupuesto.softwareseguro.com.ar/api/gastos/1/editar/>

{"monto":1500.00}

<https://chl-3b0862a5-bc83-4f06-9d7c-32ba62110b25-presupuesto.softwareseguro.com.ar/api/gastos/2/editar/>

{"monto":13000.00}

<https://chl-3b0862a5-bc83-4f06-9d7c-32ba62110b25-presupuesto.softwareseguro.com.ar/api/gastos/4/editar/>

{"monto":500.00}

<https://chl-3b0862a5-bc83-4f06-9d7c-32ba62110b25-presupuesto.softwareseguro.com.ar/api/gastos/10/editar/>

{"monto":12000.00}

<https://chl-3b0862a5-bc83-4f06-9d7c-32ba62110b25-presupuesto.softwareseguro.com.ar/api/gastos/11/editar/>

{"monto":500.00}

<img src="media/image1.png"
style="width:6.5in;height:3.85972in" />
