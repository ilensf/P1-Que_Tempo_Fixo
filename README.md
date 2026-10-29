# P1. Que tempo fixo?

Este proxecto constrúe un *pipeline* de datos meteorolóxicos diarios da
estación A Coruña / Alvedro. Descarga un ficheiro CSV, inspecciona os seus
problemas de formato e calidade, limpa os datos, responde preguntas con
Pandas, crea gráficas, compara dúas formas de calcular a amplitude térmica e
carga os datos nunha base de datos PostgreSQL para consultalos con SQL.

O notebook principal é [`notebooks/scripts.ipynb`](./notebooks/scripts.ipynb).
Debe executarse de principio a fin para actualizar as táboas e gráficas que
se mostran nas súas saídas.

## Datos e procedencia

| Campo | Información |
|---|---|
| Estación | A Coruña / Alvedro (aeroporto) |
| Indicativo | `08002` |
| Coordenadas | 43,3000° N, -8,3833° O |
| Altitude | 97 m |
| Período | 01/01/2021–31/12/2025 |
| Fonte orixinal | [Meteostat](https://meteostat.net/) |
| Ficheiro desta práctica | [CSV no repositorio de datos](https://github.com/bigdatawirtz/datasets/blob/main/datasets/meteo_alvedro_2021_2025.csv) |
| URL de descarga | [CSV directo](https://raw.githubusercontent.com/bigdatawirtz/datasets/main/datasets/meteo_alvedro_2021_2025.csv) |
| Última descarga comprobada | 29-10-2026 |

O CSV procede de Meteostat, pero foi modificado con fins didácticos para
introducir problemas de calidade. Polo tanto, non debe interpretarse como
unha descarga intacta nin como un rexistro meteorolóxico oficial. Ao executar
o notebook, o ficheiro descárgase de novo e gárdase tal como chega en
`datos/raw/`.

Para a atribución e as condicións de uso, consulta os
[termos de Meteostat](https://meteostat.net/en/terms) e as condicións que
poidan corresponder ao ficheiro educativo e ás súas modificacións.

## Requisitos

- Git.
- Python 3 e `pip`.
- Visual Studio Code coas extensións Python e Jupyter, ou outro contorno
  compatible con notebooks.
- Docker e Docker Compose para executar PostgreSQL.
- Conexión a Internet para descargar as dependencias, a imaxe de PostgreSQL
  e o CSV.

## Como executar o proxecto

Os seguintes comandos parten da raíz do repositorio.

### 1. Crear un contorno e instalar dependencias

En Linux ou macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

En Windows PowerShell:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

### 2. Configurar PostgreSQL

Crear o ficheiro local `.env` a partir do exemplo:

```bash
cp .env.example .env
```

En Windows pódese copiar `.env.example` manualmente como `.env`. Edita o
ficheiro e define un usuario e un contrasinal locais. As variables que usa
Docker Compose e o notebook son:

```dotenv
POSTGRES_USER=o_teu_usuario
POSTGRES_PASSWORD=o_teu_contrasinal
POSTGRES_DB=meteo_db
POSTGRES_PORT=5433
```

Non subas `.env` ao repositorio: está excluído por `.gitignore`. O ficheiro
`.env.example` é só unha plantilla.

Inicia o servizo desde a raíz:

```bash
docker compose up -d
docker compose ps
```

O servizo publica PostgreSQL no porto local indicado por `POSTGRES_PORT`
(5433 no exemplo) e usa o porto 5432 dentro do contedor. Antes de continuar,
comproba que o contedor aparece activo. Para ver os rexistros se non inicia:

```bash
docker compose logs postgres
```

### 3. Executar o notebook

Abre `notebooks/scripts.ipynb`, selecciona o intérprete Python do
contorno virtual e executa todas as celas en orde. As rutas do notebook son
relativas ao cartafol `notebooks/`; configura o directorio de traballo do
kernel nese cartafol. Isto permite que a descarga, o ficheiro `.env` e os
ficheiros de saída se localicen correctamente.

O notebook:

1. Descarga o CSV e gárdao sen modificalo en `datos/raw/`.
2. Inspecciona a codificación, os metadatos, os separadores, as datas e os
   códigos de valores ausentes.
3. Limpa os datos e garda o resultado en
   `datos/clean/meteo_alvedro_2021_2025_limpo.csv`, en UTF-8.
4. Calcula estatísticas e debuxa unha gráfica previa á limpeza e tres gráficas
   dos datos limpos.
5. Compara o tempo de cálculo da amplitude térmica cun bucle e cunha
   operación vectorizada, tamén cun DataFrame cinco veces maior.
6. Conéctase a PostgreSQL, crea/carga a táboa `meteo_alvedro` con `to_sql()` e
   executa tres consultas SQL.

Se se volve executar a cela de carga, `if_exists="replace"` substitúe a táboa
anterior e crea unha nova cos datos da execución actual. PostgreSQL debe estar
activo antes de executar as celas de conexión e consulta.

Ao rematar, pódese deter o servizo sen eliminar os datos persistidos:

```bash
docker compose down
```

Non engadas `-v` se queres conservar o volume da base de datos.

## Estrutura do proxecto

```text
.
├── .env.example
├── .gitignore
├── compose.yml
├── datos/
│   ├── raw/
│   └── clean/
├── notebooks/
│   └── scripts.ipynb
├── output/
├── README.md
└── requirements.txt
```

Os CSV de `datos/raw/` e `datos/clean/` xéranse ao executar o notebook. Para
a entrega, mantén baleiros eses cartafoles no repositorio e conserva neles
unicamente os ficheiros `.gitkeep` se son necesarios para representar os
cartafoles.

## Lectura e decisións de limpeza

O ficheiro ten seis liñas de metadatos antes da cabeceira, usa `;` como
separador, coma decimal e contén códigos de ausencia como `NA` e `-9999`.
Tamén aparece `Ip` en precipitación. O notebook inspecciona os bytes e as
liñas antes de ler a táboa con Pandas.

| Problema | Detección | Tratamento |
|---|---|---|
| Metadatos antes da cabeceira e codificación | Inspección dos bytes e das primeiras e últimas liñas: a cabeceira identifica CP1252 | Saltar as seis liñas de metadatos. A lectura actual usa `latin-1`; convén manter este parámetro coherente coa codificación identificada. A saída gárdase en UTF-8. |
| Separador e decimais non estándar | Inspección do CSV e dos tipos inferidos | Ler con `sep=";"` e converter os valores numéricos admitindo coma decimal e punto de milleiros. |
| Códigos `NA`, `-9999` e campos baleiros | Revisión dos valores das columnas | Convertelos en valores ausentes (`NaN`), sen inventar medidas. |
| Código `Ip` en precipitación | Revisión dos valores orixinais da columna | Interpretalo como 0 mm para os cálculos de precipitación. |
| Datas en dous formatos | Revisión da columna `Data` | Interpretar tanto `AAAA-MM-DD` como `DD/MM/AAAA`; as datas non interpretables quedan contabilizadas. |
| Duplicados e datos fóra de orde | Contaxe de datas repetidas e comparación coa orde temporal | Ordenar por data e conservar a primeira fila de cada data. |
| Días sen rexistro | Comparación co calendario diario entre 2021 e 2025 | Detectáronse 27 datas ausentes. Interpólanse linealmente as variables distintas da precipitación nos ocos interiores de ata 7 días; os ocos de máis de 7 días quedan nulos (incluído o intervalo do 3 ao 23 de xullo de 2023). Os valores ausentes de filas orixinais tampouco se interpolan. **A precipitación nunca se interpola.** |
| Temperaturas sospeitosas ou incompatibles | Revisión de limiares estacionais e comparación entre `tmin`, `tmed` e `tmax` | Anular a medida afectada cando a máxima supera 35 °C fóra de xuño–agosto, a mínima é negativa entre xuño e setembro, ou a media queda fóra do intervalo mínima–máxima. Son regras heurísticas documentadas no notebook. |
| Precipitación negativa ou ausente | Conversión numérica e revisión do rango | Manter os datos ausentes como nulos; non substituílos por cero. |

Tras completar o calendario hai 1.826 datas. A interpolación aplícase só ás
datas que faltaban no ficheiro e que forman ocos interiores de ata 7 días;
non se usa para a precipitación, para ocos máis longos nin para valores
ausentes de filas que xa existían. **A precipitación nunca se interpola**, así
non se inventan datos de choiva.
As análises de precipitación exclúen os meses nos que falta algunha
observación, para non contar un día descoñecido como día seco.

## Análise, gráficas e PostgreSQL

O notebook responde as preguntas do apartado 4.4:

- Calcula a temperatura media anual. Os resultados gardados son 14,10 °C
  (2021), 14,98 °C (2022), 14,91 °C (2023), 14,44 °C (2024) e 14,45 °C
  (2025). A diferenza entre o primeiro e o último ano é +0,35 °C, pero cinco
  anos non abondan para concluír unha tendencia climática.
- Calcula a precipitación media mensual a partir dos totais mensuais. Con
  meses completos, novembro é o máis chuvioso (164,6 mm de media) e agosto o
  máis seco (15,4 mm).
- Conta como chuvioso un día con precipitación **igual ou superior a 1 mm**.
  Só se inclúen meses completos; novembro ten a maior media de días chuviosos
  (17,2).
- Busca o día con maior `tmax` e menor `tmin` en cada ano.

Hai unha gráfica da temperatura máxima antes da limpeza e tres gráficas dos
datos limpos: temperatura media diaria coa media móbil de 30 días,
precipitación media mensual e comparación de 2025 coa media de 2021–2024.

Para medir a eficiencia, o notebook calcula `tmax - tmin` con `iterrows()` e
con operación vectorizada. Nas saídas gardadas, para o tamaño orixinal as
medicións son aproximadamente 22,3 ms fronte a 23,3 μs; para cinco veces os
datos, 110 ms fronte a 26,7 μs. Os tempos poden variar segundo o equipo.

Na parte PostgreSQL, `to_sql()` carga 1.826 filas. As consultas calculan a
temperatura media anual, a precipitación media por mes e o maior vento medio
rexistrado en cada ano. A temperatura media anual coincide cos resultados de
Pandas. Na precipitación, os meses máis e menos chuviosos tamén coinciden;
febreiro pode aparecer como 78,8 mm en Pandas e 78,9 mm en PostgreSQL por
diferencias na regra de redondeo cando o valor queda xusto no punto medio.
Compara os valores sen redondear para comprobar a igualdade exacta.

## Uso da intelixencia artificial

Empregouse Copilot en VS Code como axuda para revisar o código do notebook,
analizar erros de execución e redactar esta documentación. Os resultados e
decisións de limpeza deben comprobarse executando o notebook; a ferramenta
non substitúe a validación dos datos.
