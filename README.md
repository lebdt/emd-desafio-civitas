# Proposta

## Raciocínio

Seria impossível percorrer a distância em uma arco?reta de um radar ao outro no tempo considerando a velocidade do carro no momento da leitura até o próximo radar. O carro deve estar submetido às leis da física e se adicionarmos isso ao fato de que é pouco provável que o carro siga o caminho 'físico' de menor distância.

Exemplo:

<p align="center">
    <img src="https://imgur.com/a/U0T5xc9"/>
</p>

Aqui vemos um trajeto entre dois radares de coordenadas `(-22.883354,-43.237033)` e `(-22.924788,-43.389069)` com tempo médio estimado via Google Maps. 

Calculando a distância física aproximada entre esses dois pontos na superfícia da terra, temos como resultado $d_F = 16.24\,\mathrm{km}$ e supondo que percorreu com uma velocidade constante de $100\,\mathrm{km/h}$ sem obstáculos e em linha reta o tempo estimado está em torno de 9 minutos e 45 segundos. Qualquer tempo abaixo disso deve ser destacado como possível clonagem.

Porém é improvável que um percurso ocorra em velocidade constante. Parte do desafio consiste em lidar com esse fato.
<!--e seguindo as vias ainda sem obstáculos esse tempo deve ser 13 minutos e 30 segundos.-->

## Necessário

Conversão de distância entre coordenadas e coerência na comparação com a velocidade e tempo. A priori faz sentido desconsiderar quaisquer possíveis curvaturas e elevações; Radares distintos;

## Questionamentos

O que fazer com coordenadas fora dos parâmetros plausíveis? E.g. 0, 0 ou coordenadas claramente fora do escopo?

*IMPORTANTE: Determinar se os dados estão de acordo com parâmetros reais*

Contudo, mesmo que não sejam, a lógica de que o critério para se produzir uma suspeita de clonagem de placa deve ser baseado no tempo que se levaria a percorrer até o próximo radar em que a placa também é identificada persiste. 


# Análise a priori

## Identificação das colunas/features de interesse

`datahora`, `velocidade`, `placa`, `camera_latitude`, `camera_longitude`


## Checagem Elementos Nulos

```sql
SELECT *
FROM `rj-cetrio.desafio.readings_2024_06`
WHERE placa IS NULL
OR camera_latitude IS NULL
OR camera_longitude IS NULL
OR datahora IS NULL
OR velocidade IS NULL
```
Output: []

## Máximos e mínimos

Nota-se a presença das coordenadas de latitude 0 e longitude 0 assim como velocidade 0

```sql
SELECT
  min(camera_latitude) min_lat,
  max(camera_latitude) max_lat,
  min(camera_longitude) min_lon,
  max(camera_longitude) max_lon,
  min(velocidade) min_vel,
  max(velocidade) max_vel
FROM `rj-cetrio.desafio.readings_2024_06`
```
Output:

| min_lat | max_lat | min_lon | max_lon | min_vel | max_vel |
| :---: | :---: | :---: | :---: | :---: | :---: |
| -23.8597222 | 0.0 | -43.690229 | 43.334218 | 0 | 255 |


Apenas com a aba de visualização dos dados brutos foi possível observar alguns padrões, como é o caso da velocidade estar no formato de quilômetros por hora (km/h) além de, como relatado no desafio, ser do tipo `INTEGER`, o que se provou vantajoso no futuro para calcular o valor máximo entre dois fatores; valores de latitudes e longitudes dados em graus.

E como a informação disponível diz respeito apenas à velocidade, não há como considerar as variações de velocidade dentro dos trajetos (levando-se em conta as vias) de menor caminho. O limiar de plausibilidade, aqui também tratado como tempo mínimo, deve conter apenas o registro de velocidade em ambos os radares comparados, onde a velocidade deve ser a máxima entre os dois comparados e qualquer tempo de percurso menor do que o tempo levado na velocidade máxima deve ser tratado como pouco provável.

A velocidade máxima é a mais indicada nesse caso para assegurar que o limiar de tempo plausível seja o menor possível, onde provavelmente qualquer tempo de percurso entre dois radares menor do que o determinado por esse método deverá representar uma possível clonagem de placa.


## Possíveis Abordagens para Cálculo da Distância

Aproximação para o raio da Terra. Raio médio $R = 6371\,\mathrm{km}$

1. **Método da distância geográfica (Leva em conta a elipticidade da Terra)**

2. **Método de Haversine (esfera perfeita)**

3. **Método personalizado**

O método escolhido na verdade é o método para calcular distâncias em um plano, considerando latitude e longitude como coordenadas nesse plano acrescidas de uma constante que, com o ângulo (latitude/longitude) produz arco de circunferência.

Nesse caso $\piR / 180^{\circ}$, onde $\pi$ foi aproximado para $3.14$.

$$
\frac{\pi R}{180^{\circ}}\sqrt{(lat_1 - lat_2)^2 + (lon1 - lon2)^2}
$$

Apesar de ser um método 'grosseiro' a primeira vista, as distâncias são minúsculas em relação à circuferência terrestre, ainda é o método que dispensa o uso de funções trigonométricas que pode torná-lo mais legível e como estamos lindando com um conjunto de centenas de milhões de linhas, qualquer pequena otimização pode ser vantajosa em uma consulta.

Por meio de alguns testes dentro das coordenadas do estado do Rio de Janeiro. o erro em relação ao resultado de maior precisão (Método 1) obtido esteve no seguinte.

$ 0.02 < \varepsilon < 0.09 $


# Query Principal

```sql
CREATE TEMP FUNCTION max_int(x INT64, y INT64)
RETURNS FLOAT64
AS (
  (ABS(x - y) + x + y) * 1/2
);

CREATE TEMP FUNCTION distance(lat1 FLOAT64, lon1 FLOAT64, lat2 FLOAT64, lon2 FLOAT64)
RETURNS FLOAT64
AS (
  3.14*6371/180 * sqrt((lat1 - lat2) * (lat1 - lat2) + (lon1 - lon2) * (lon1 - lon2))
);

CREATE TEMP FUNCTION delta_time(end_time TIMESTAMP, start_time TIMESTAMP)
RETURNS FLOAT64
AS (
    ABS(TIMESTAMP_DIFF(end_time, start_time, SECOND))* 1/3600
);

SELECT DISTINCT
  r1.placa
FROM
  `rj-cetrio.desafio.readings_2024_06` AS r1
JOIN
  `rj-cetrio.desafio.readings_2024_06` AS r2
  ON r1.placa = r2.placa
  AND r1.datahora < r2.datahora
  AND (r1.velocidade <> 0 OR r2.velocidade <> 0)
  WHERE delta_time(r1.datahora, r2.datahora) < distance(r1.camera_latitude, r1.camera_longitude, r2.camera_latitude, r2.camera_longitude) / max_int(r1.velocidade, r2.velocidade)
```


# Queries Alternativas

Outras queries que pode trazer insights sobre as detecções é a que identifica quais radares e horários associados às placas com possível clonagem.

*Todas as queries a seguir são apresentadas sem o 'cabeçalho' contendo a definição das funções a fim de evitar repetições*

```sql
SELECT
  r1.placa,
  r1.datahora,
  r2.datahora AS datahora_2,
  delta_time(r1.datahora, r2.datahora) AS act_time,
  distance(r1.camera_latitude, r1.camera_longitude, r2.camera_latitude, r2.camera_longitude) / max_int(r1.velocidade, r2.velocidade) AS min_time
FROM
  `rj-cetrio.desafio.readings_2024_06` AS r1
JOIN
  `rj-cetrio.desafio.readings_2024_06` AS r2
  ON r1.placa = r2.placa
  AND r1.datahora < r2.datahora
  AND (r1.velocidade <> 0 OR r2.velocidade <> 0)
  WHERE delta_time(r1.datahora, r2.datahora) < distance(r1.camera_latitude, r1.camera_longitude, r2.camera_latitude, r2.camera_longitude) / max_int(r1.velocidade, r2.velocidade)
```

A que determina quantas vezes são acusadas como possivelmente clonadas, onde, quanto maior a contagem, maior a probabilidade dessa placa ter sido clonada. E com uma análise a posteriori, pode-se definir um limite de contagens para falsos positivos.

```sql
SELECT placa, count(datahora) AS contagem FROM (
  SELECT
    r1.placa,
    r1.datahora,
    r2.datahora AS datahora_2,
    delta_time(r1.datahora, r2.datahora) AS act_time,
    distance(r1.camera_latitude, r1.camera_longitude, r2.camera_latitude, r2.camera_longitude) / max_int(r1.velocidade, r2.velocidade) AS min_time
  FROM
    `rj-cetrio.desafio.readings_2024_06` AS r1
  JOIN
    `rj-cetrio.desafio.readings_2024_06` AS r2
    ON r1.placa = r2.placa
    AND r1.datahora < r2.datahora
    AND (r1.velocidade <> 0 OR r2.velocidade <> 0)
    WHERE delta_time(r1.datahora, r2.datahora) < distance(r1.camera_latitude, r1.camera_longitude, r2.camera_latitude, r2.camera_longitude) / max_int(r1.velocidade, r2.velocidade)
)
GROUP BY placa
```


E também a que aponta a quantidade de placas e total de placas detectadas

```sql
SELECT
  (
    SELECT COUNT(*) FROM (
      SELECT DISTINCT
        r1.placa
      FROM
        `rj-cetrio.desafio.readings_2024_06` AS r1
      JOIN
        `rj-cetrio.desafio.readings_2024_06` AS r2
        ON r1.placa = r2.placa
        AND r1.datahora < r2.datahora
        AND (r1.velocidade <> 0 OR r2.velocidade <> 0)
        WHERE delta_time(r1.datahora, r2.datahora) < distance(r1.camera_latitude, r1.camera_longitude, r2.camera_latitude, r2.camera_longitude) / max_int(r1.velocidade, r2.velocidade)
    )
  ) AS t_clone,
  (
    SELECT COUNT(DISTINCT rt.placa) FROM `rj-cetrio.desafio.readings_2024_06` as rt
  ) AS t_placa,
```

Exemplo de output das primeiras 10,000 linhas do conjunto de dados. O resultado obtido foi:

| t_clone | t_placa |
|:-------:|:-------:|
| 46      | 7981    | 


<!--## Teste com conjunto de dados simulado-->


## Considerações

O método está longe de ser perfeito. Levando em conta a função avaliada para determinar possível clonagem 'distancia', tempos distintos marcados pelo mesmo radar são desconsiderados visto que não há como encontrar resultado menor do que 0, e isso permite que carros de placas clonadas que realizando o mesmo percurso dos de placas originais não sejam detectados.
