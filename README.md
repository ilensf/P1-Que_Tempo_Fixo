# Datos meteorolóxicos de A Coruña / Alvedro

Pipeline de datos diarios da estación aeroportuaria A Coruña / Alvedro
(indicativo **08002**), para o período 2021–2025. As coordenadas que aparecen
na cabeceira do ficheiro son **43,3000° N, -8,3833° O**, altitude **97 m**.

O ficheiro procede de Meteostat e simula unha exportación dun portal de datos;
inclúe problemas de calidade introducidos con fins didácticos. Non se debe
tratar como unha fonte meteorolóxica oficial nin como unha exportación intacta
de Meteostat.

## Fonte e uso

- Fonte orixinal: [Meteostat](https://meteostat.net/).
- Ficheiro usado no proxecto:
  [CSV publicado](https://raw.githubusercontent.com/bigdatawirtz/datasets/main/datasets/meteo_alvedro_2021_2025.csv).
- O ficheiro contén a atribución a Meteostat e indica que foi modificado con
  fins didácticos. Non se especifica unha licenza para esta copia; antes de
  redistribuíla, consulta os [termos de Meteostat](https://meteostat.net/en/terms).
- Verificación da URL: **29-10-2026**. A URL devolveu 81.825 bytes, idénticos
  byte a byte ao ficheiro local. Ao executar o notebook, descárgase de novo o
  CSV e gárdase en bytes en `datos/raw/`.

## Execución

1. Instala Python e crea/activa un contorno virtual.
2. Instala as dependencias:

   ```bash
   python -m pip install -r requirements.txt
   ```

3. Abre `notebooks/scripts.ipynb` e executa todas as celas en orde co cartafol
   de traballo `notebooks/`. Se `datos/raw/` está baleiro, a primeira cela
   descarga o CSV. O ficheiro raw gárdase en bytes sen reinterpretalo nin
   modificalo.
4. As celas seguintes inspeccionan os bytes e o formato, len e limpan os datos,
   gardan `datos/clean/meteo_alvedro_2021_2025_limpo.csv` en UTF-8, calculan
   as respostas do apartado 4.4 e xeran as gráficas.

O notebook debe executarse desde o principio para actualizar as saídas visibles.
A medida de eficiencia e PostgreSQL aínda están pendentes de completar.

## Decisións de lectura e limpeza

| Problema observado | Detección | Decisión |
|---|---|---|
| Codificación CP1252, seis liñas previas á cabeceira e separador `;` | Inspección dos bytes e das primeiras/últimas liñas antes de pandas | Ler con `encoding="cp1252"` e saltar as seis liñas; conservar `datos/raw/` sen cambios |
| Decimais con coma, presión con punto de milleiros e valores con punto decimal nalgúns rexistros | Comparación de tipos inferidos e inspección dos valores orixinais | Normalizar cada valor numérico admitindo ambas as convencións antes de convertelo |
| Códigos `NA`, `-9999` e campos baleiros | Revisión de valores orixinais por columna: 12 ausencias en cada temperatura, 10 en precipitación e 1 en presión | Tratalos como descoñecidos (`NaN`), non como medidas reais |
| `Ip` en precipitación (precipitación inapreciable) | Inspección dos códigos da columna de precipitación: 121 aparicións | Convertelo a 0 mm para sumas e para o criterio de día chuvioso (≥ 1 mm); non chega ao limiar |
| Dúas formas de data (`AAAA-MM-DD` e `DD/MM/AAAA`) | Inspección das cadeas: 164 datas usan `DD/MM/AAAA` | Parsear cada formato co seu patrón propio; non aplicar `dayfirst` ás datas ISO |
| Datas duplicadas | Agrupación pola data xa parseada: 13 filas repetidas en 13 datas, 5 con medidas distintas | Conservar a primeira aparición do ficheiro e eliminar as repeticións posteriores; non sumar rexistros duplicados |
| Días sen rexistro e rexistros fóra de orde | Comparación co calendario diario completo: faltan 27 días; detéctanse 9 descensos na orde de datas | Ordenar e reindexar para incluír todos os días; deixar as medidas ausentes como nulas, sen interpolar |
| Oco longo de xullo de 2023 | Identificación do tramo ausente do 3 ao 23 de xullo (21 días) | Manter as medidas como nulas; non inventar unha serie para o tramo |
| Valores meteorolóxicos imposibles ou sospeitosos | Detección de sentinelas; unha media fóra de `[tmin, tmax]`; tres `tmax > 35 °C` fóra de xuño–agosto e unha `tmin < 0 °C` entre xuño e setembro | Anular só a medida afectada. Os limiares estacionais son unha revisión heurística, non unha lei física |
| Precipitación ausente ou negativa | Conversión numérica e comprobación do rango | Deixala como descoñecida; non substituíla por 0 nin interpolala |

As medias e extremos de temperatura usan os valores dispoñibles tras esta
limpeza. Para comparar precipitación mensual e días chuviosos só se usan meses
con precipitación coñecida en todos os días (44 dos 60 meses); así non se
interpreta un día descoñecido como día seco. O número de meses completos pode
variar segundo o mes do ano.

## Uso da IA

Empregouse Copilot en VS Code para revisar a estrutura do proxecto, detectar
erros de lectura e limpeza, e axudar a corrixir o notebook e esta
documentación. As decisións e os resultados deben revisarse e executarse no
notebook antes da entrega.
