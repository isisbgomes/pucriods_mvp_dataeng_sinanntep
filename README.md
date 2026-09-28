_pucriods_mvp_dataeng_sinanntep_

# Projeto de MVP de Engenharia de Dados usando o Sistema Nacional de Agravos de Notificação Obrigatória (SINAN) vs. Nexo Técnico Epidemiológico Previdenciário (NTEP)

O objetivo deste MVP é criar um pipeline de dados que permita avaliar a notificação no SUS de dados de doenças e condições de notificação obrigatória, entre acidentes de trabalho e outras condições previstas no Nexo Técnico Epidemiológico Previdenciário (NTEP).

**Contexto**
O NTEP é um cruzamento de dados de doenças e condições ocupacionais ou relacionadas ao trabalho que facilita a identificação de agravos acidentários, a partir da associação entre atividades econômicas e risco de adoecimento de trabalhadores.

O NTEP existe desde 2007, quando a Lei 11.430/2006 entrou em vigor, e é operacionalizado pelo cruzamento do CNAE (Cadastro Nacional de Atividades Econômicas) com dados de CID (Classificação Internacional de Doenças). É atualizado por meio de decreto — o mais atual é o Decreto n. 6042 de 2007. O NTEP dispensa a apresentação de Comunicado de Acidente de Trabalho para a concessão de benefícios de natureza acidentária no contexto de trabalho, que, entre outros benefícios, garantem estabilidade por até 12 meses após a volta ao trabalho e o depósito de FGTS pelo empregador.

O SINAN é um sistema de notificações geral do Ministério da Saúde, que coleta dados de doenças e condições de interesse de todos os estabelecimentos de assistência à saúde do Brasil, com regularidade substancial, variável por condição, com no máximo 60 dias de atraso da identificação da condição. As secretarias municipais e estaduais de saúde são obrigadas legalmente a apresentar dados dessa natureza ao Ministério da Saúde. A lista atualizada de condições que o SINAN registra foi definido pela PRT MS/GM 204/2016, Anexo 1.

**Objetivo principal**
Cruzar os dados do SINAN com o NTEP permite avaliar o nível de adoecimento limite geral da população e estabelecer um teto hipotético para notificações acidentais, caso todas as notificações registradas no sistema de saúde ocorressem por razões associadas ao trabalho e seu ambiente. Análises nessa linha poderiam incluir dados demográficos sobre a população acometida, como sexo e raça, para mapear vulnerabilidades específicas.

**Objetivo Secundário**
Outra pergunta que poderia ser respondida seria se o NTEP precisa ser atualizado e confrontado com as práticas de registro de agravos obrigatórios no sistema de saúde, dado que a lista de definições do SINAN é de 2016. 

Objetivo secundário 1. Por exemplo, o NTEP prevê associação de uma única arbovirose, Dengue, com alguns CNAES, mas não de outras. Dado que o vetor da doença é o mesmo e a forma de exposição seria relacionada ao trabalho, o NTEP poderia ser ajustado para incluir também Chikungunya e Zika associado às mesmas ocupações que ampliam o risco para Dengue. Os dados de Chikungunya e Zika constam no SINAN, mas não no NTEP.

Objetivo secundário 2. Outro exemplo seria caso o Decreto registrasse códigos CID que estivessem em discordância ou não gerassem notificações sensíveis, que pudessem ser cruzadas com os sistemas previdenciários, como indicador sentinela de acidentes ou agravos relacionados ao trabalho.  

**Coletando Dados do FTP do SINAN**
**Importação de forma paralelizada**

- Importação de dados .dbc do ftp datasus que arquivam documentos do SINAN desde 2008, caso estejam disponíveis (adotamos um ano de lag de implementação do sistema)
- Download de tabelas com 18 agravos de interesse do NTEP:  "DENGUE", "CHIKUNGUNYA", "ZIKA", "TUBERCULOSE", "MENINGITE",
    "ACIDENTE DE TRABALHO COM MATERIAL BIOLOGICO", "VIOLENCIA", "HEPATITES", "CANCER RELACIONADO AO TRABALHO", "DERMATOSE RELACIONADO AO TRABALHO", "INTOXICACAO EXOGENA", "LEPTOSPIROSE", "LER/DORT", "MALARIA", "PAIR", "PNEUMOCONIOSE", "TETANO ACIDENTAL", "TRANSTORNO MENTAL RELACIONADO AO TRABALHO".
- Observação 1: as condições de Chikungunya e de Zika foram adicionadas por dedução à lista de interesse — elas não estão previstas na última atualização do NTEP em 2007, embora as doenças tenham se tornado condições de interesse de saúde pública a partir de 2016 e sejam transmitidas pelo mesmo vetor da dengue, que está no NTEP. Observação 2: Foram incluídos acidente de trabalho com material biológico (ACBI), mas excluídos Acidente de trabalho sem especificação (ACGR). Não foi possível incluir Leishmaniose, pela subcategorização feita pelo SINAN.
- Depois, os dados foram descompactados em .dbf, parseados, categorizados e salvos em formato parquet (apenas dados das colunas de interesse, para evitar que o sistema caísse por capacidade de processamento. 

As colunas de interesse, agrupadas por camada de processamento, são:

CAMADA BRONZE: Foi pulada, porque os arquivos .dbf do ftp DATASUS têm às vezes +1,6M de linhas, e no ambiente databricks gratuito, isso estava causando muitas quedas e interrupções no script. Assim, o mapeamento de colunas e códigos e derivação de campos foi feita já na ingestão.

CAMADA SILVER: Inclui colunas harmonizadas em todas as tabelas, com
- "NU_ANO": ano da notificação
- "DT_NOTIFIC": data da notificação
- "ID_MUNICIP": código de identificação do município de notificação
- "ID_MN_RESI": código de identificação do município de residência da pessoa
- "ID_AGRAVO": código do agravo
- "CID": código da Classificação Internacional de Doença
- "CS_SEXO": classificação de sexo da pessoa
- "CS_RACA": classificação da raça da pessoa
- "CAT": categoria de acidente de trabalho

CAMADA GOLD: Inclui dados derivados
- Enriquecimento por categoria de Capítulo de CID e Categoria de interesse do NTEP (via join com o dicionário de dados json gerado antes), com filtragem

**Categorias de análise**
Após criação de tabelas em esquema estrela do tipo dimensão (de notificação, de local de notificação, e de pessoa notificadora) mais tabela a fato resultante dessas dimensões, foram analisados aspectos descritivos básicos como:
1. Aspectos de risco: Categorias do NTEP mais notificadas, em série histórica
2. Aspectos de regionalidade: UF, Região, 
3. Aspectos da pessoa: Sexo (Feminino, Masculino, Em Branco, Ignorado) e Raça (Ign/Branco, Amarela, Branca, Parta, Preta, Indígena, Ignorado)

**Estruturação da Base de Dados e do ETL**
Foram criadas 3 tabelas dimensão (notificação, notificante, local de notificação), e uma fato com chave PK sintética.
A documentação dos campos dessas tabelas foi realizada nas próprias Delta Table, bem como das tabelas silver e gold (ver notebook z. Documentação). 

**Investigação e Resposta aos objetivos**
A investigação da qualidade dos dados foi feita no notebook 2, avaliando as categorias de agravos perdidas da camada Silver para a Gold. 
A resposta aos objetivos listados acima foi dada nos notebooks 3 e 4.

**Autoavaliação**
A autoavaliação foi feita no notebook 4. Análise.
