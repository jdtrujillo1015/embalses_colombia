# Embalse El Peñol-Guatapé — Monitor de Volumen

Sistema de predicción del volumen útil del embalse El Peñol-Guatapé (Colombia) con despliegue automático en AWS.

**Dashboard en vivo:** [https://d12u2i9a3oakm0.cloudfront.net](https://d12u2i9a3oakm0.cloudfront.net)

---

## Descripción

El embalse El Peñol-Guatapé es el principal activo de generación hidroeléctrica de Colombia, con una capacidad útil de ~1.100 Mm³. Su volumen determina el despacho de energía del Sistema Interconectado Nacional (SIN), el abastecimiento de agua del oriente antioqueño y la operación de compuertas de Empresas Públicas de Medellín (EPM).

Este proyecto construye un pipeline de MLOps que ingesta datos diarios de la API de XM (operador del mercado energético colombiano), entrena cuatro modelos de series de tiempo y publica pronósticos a 7, 15 y 30 días en un dashboard público actualizado automáticamente.

---

## Arquitectura

```
API XM (pydataxm)
      │
      ▼
SageMaker Pipeline (11 pasos)
      │
      ├── Ingesta ──► S3 raw (Parquet particionado por año/mes)
      │
      ├── Procesamiento ──► S3 curated (serie diaria limpia)
      │
      ├── Entrenamiento ARIMA ──► S3 models/arima/
      ├── Inferencia ARIMA ──────► S3 predictions/arima/
      │
      ├── Entrenamiento ARMA-GARCH ──► S3 models/garch/
      ├── Inferencia ARMA-GARCH ─────► S3 predictions/garch/
      │
      ├── Entrenamiento Holt-Winters ──► S3 models/hw/
      ├── Inferencia Holt-Winters ─────► S3 predictions/hw/
      │
      ├── Entrenamiento LSTM ──► S3 models/lstm/
      ├── Inferencia LSTM ─────► S3 predictions/lstm/
      │
      └── Dashboard ──► S3 + CloudFront (HTML estático)

EventBridge (cron 17:00 UTC) ──► dispara pipeline diariamente
SNS ──► notificación email al completar o fallar
```

---

## Modelos

| Modelo | MAPE test | MAE test | RMSE test |
|--------|-----------|----------|-----------|
| Holt-Winters ETS | **0.57%** | **2.55 Mm³** | **4.06 Mm³** |
| LSTM (2 capas) | 0.95% | 5.92 Mm³ | 7.51 Mm³ |
| ARMA-GARCH(1,1) | 6.18% | — | — |
| ARIMA | 18.12% | — | — |

Split cronológico 80/20 sobre ~4.200 observaciones diarias (2015–2026). Métricas OOS calculadas sobre los primeros 30 días del conjunto de test (horizonte operativo real).

**Holt-Winters supera al LSTM** porque la serie tiene estacionalidad anual marcada y tendencia suave — estructura que ETS modela explícitamente con `seasonal_periods=365`, mientras que el LSTM con ventana de 30 días no alcanza a capturar el ciclo completo.

---

## Stack tecnológico

| Capa | Tecnología |
|------|------------|
| Orquestación ML | AWS SageMaker Pipelines |
| Cómputo | SageMaker Processing Jobs (ml.t3.medium) |
| Almacenamiento | Amazon S3 |
| Automatización | Amazon EventBridge |
| Notificaciones | Amazon SNS |
| Dashboard | HTML/JS estático en S3 + CloudFront |
| Datos | API pública XM (`pydataxm`) |
| Modelos | pmdarima, arch, statsmodels, TensorFlow/Keras |

---

## Estructura del repositorio

```
embalses_colombia/
├── scripts/                        # Processing Jobs del pipeline
│   ├── ingesta_guatape.py          # Descarga incremental desde API XM
│   ├── procesamiento_guatape.py    # Limpieza e imputación
│   ├── entrenamiento_arima.py
│   ├── inferencia_arima.py
│   ├── entrenamiento_garch.py
│   ├── inferencia_garch.py
│   ├── entrenamiento_holtwinters.py
│   ├── entrenamiento_hw.py
│   ├── inferencia_hw.py
│   ├── entrenamiento_lstm.py
│   ├── inferencia_lstm.py
│   └── dashboard.py                # Genera el HTML del dashboard
├── notebooks/
│   ├── pipeline_orquestador.ipynb  # Registra y lanza el pipeline
│   ├── pipeline_orquestador_final.ipynb
│   └── EDA.ipynb                   # Análisis exploratorio
└── README.md
```

---

## Despliegue

### Requisitos
- Cuenta AWS con SageMaker, S3, EventBridge y SNS habilitados
- Rol IAM con permisos sobre estos servicios
- Instancia de notebook SageMaker

### Pasos

```bash
# 1. Clonar el repositorio en la instancia de notebook
cd /home/ec2-user/SageMaker
git clone https://github.com/jdtrujillo1015/embalses_colombia.git

# 2. Subir scripts a S3
aws s3 sync embalses_colombia/scripts/ s3://<tu-bucket>/scripts/

# 3. Abrir y ejecutar pipeline_orquestador_final.ipynb
```

### Actualizar después de cambios

```bash
git pull
aws s3 sync embalses_colombia/scripts/ s3://<tu-bucket>/scripts/
# Re-ejecutar el notebook orquestador
```

---

## Fuente de datos

- **Operador:** XM S.A. E.S.P. — operador del Sistema Interconectado Nacional de Colombia
- **Variable:** `VoluUtilDiarMasa` — volumen útil diario del embalse (m³)
- **Embalse:** `PENOL` (El Peñol-Guatapé)
- **Cobertura:** 2015-01-01 hasta hoy (actualización diaria automática)
- **Acceso:** API pública gratuita vía `pydataxm`

---

## Autor

Juan David Trujillo — [github.com/jdtrujillo1015](https://github.com/jdtrujillo1015)
