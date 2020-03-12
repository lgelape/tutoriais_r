# RSelenium

*Lucas Gelape*

*Março de 2020*

Tutorial preparado para treinamento dos bolsistas do NECI-USP, 2020.

## Introdução

Vários pacotes de R nos auxiliam a raspar dados da internet. O `RSelenium` é um deles. 

As coletas de dados anteriores se basearam principalmente no pacote `rvest` que faz uso principalmente da ideia de páginas estáticas - ou seja, de que cada endereço utilizado corresponde a uma única página. Contudo, você já deve ter se deparado com as chamadas *páginas dinâmicas*, aquelas em que o conteúdo é alterado sem uma respectiva modificação no endereço da página. Ou seja, ao tentar automatizar uma coleta, você conseguiria raspar somente as informações da página inicial à qual o endereço está vinculado.

O `RSelenium` nos possibilita automatizar um navegador e utilizá-lo para coletar as informações de cada página. É como se criássemos um navegador "fantasma", controlado via R, e raspássemos as informações de cada página "dinamicamente" criada. 

Neste tutorial, vamos explorar as potencialidades do `RSelenium` ao conduzir uma coleta de dados no site da Câmara Municipal de Belo Horizonte.

## Cuidado importante

Este tutorial foi produzido em março de 2020. Pacotes (e outros programas/páginas relacionados a eles) estão em constante modificações, então funções podem deixar de funcionar ou terem comportamentos diversos ao longo do tempo.

# Etapas preparatórias

Antes de começarmos a utilizar o `RSelenium` para raspar páginas precisamos realizar algumas etapas preparatórias. Em minhas experiências, a raspagem de dados foi mais bem sucedida ao utilizar o Google Chrome. 

A versão de R e pacotes usados neste tutorial são:

```r
sessionInfo()
# R version 3.6.1 (2019-07-05)

# attached base packages:
# [1] stats     graphics  grDevices utils     datasets  methods   base     

# other attached packages:
# [1] stringr_1.4.0   RSelenium_1.7.5

# loaded via a namespace (and not attached):
# [1] compiler_3.6.1   magrittr_1.5     assertthat_0.2.1 tools_3.6.1     
# [5] wdman_0.2.4      binman_0.1.1     Rcpp_1.0.3       stringi_1.4.3   
# [9] caTools_1.17.1.2 openssl_1.4.1    bitops_1.0-6     semver_0.2.0    
# [13] XML_3.98-1.20    askpass_1.1     
```

## Chrome Driver

Para utilizarmos esse navegador, precisamos instalar o [Chrome Driver](https://sites.google.com/a/chromium.org/chromedriver/), que é uma ferramenta que permite que um driver Selenium controle um navegador Chrome. O recomendável é utilizar a última versão divulgada (e stable) da sua versão do Chrome ([mais informações sobre como identificar a versão adequada do Chrome Driver podem ser encontradas neste link, em inglês](https://sites.google.com/a/chromium.org/chromedriver/downloads/version-selection)). Na minha versão deste tutorial, estou usando a versão `75.0.3770.140` do driver. Você deve adaptar segundo aquela instalada em seu computador.

[Para não utilizarmos o Docker, podemos carregar um servidor Selenium utilizando a função `rsDriver`](https://cran.r-project.org/web/packages/RSelenium/vignettes/basics.html#introduction). Nela, precisamos especificar os argumentos `browser` (que define qual navegador utilizaremos), `chromever` (que define qual a versão do Chrome Driver utilizada) e [`port`](https://www.lifewire.com/port-numbers-on-computer-networks-817939) (aqui utilizamos `4567L`, que é o padrão da função).

Em seguida, vinculamos um `client` ao servidor que abrimos, [que vai nos permitir fazer requisições em linguagem R ao servidor que acessamos](https://www.pawangaria.com/post/automation/what-is-selenium-webdriver/). 

```r
library(RSelenium)

rD <- rsDriver(browser = c("chrome"), chromever = "75.0.3770.140",
               port = 4567L)
cliente <- rD$client
```

# Coletando projetos de lei no site da Câmara Municipal de Belo Horizonte

Para exemplificar como podemos usar o RSelenium, faremos uma coleta de informações de projetos de lei que tramitam na [Câmara Municipal de Belo Horizonte](https://www.cmbh.mg.gov.br/) (CMBH). 

Com os passos realizados anteriormente, nós abrimos uma janela do Chrome que será controlada remotamente. Será que ela está funcionando? Para testar, vamos abrir a página inicial da CMBH, com a função `$navigate`. 

```r
cliente$navigate("https://www.cmbh.mg.gov.br/")
```

Queremos extrair algumas informações sobre projetos de lei da CMBH. Essas informações podem ser obtidas na Pesquisa de Legislação, que pode ser acessada no box da página inicial. Vamos acessar a Pesquisa Avançada, para termos mais opções de informações a serem coletadas. 

Se nossa ideia é automatizar a coleta, como vamos clicar no ícone que desejamos? 

A função `$findElement()` nos ajuda a "encontrar" o elemento de uma página. Nós conseguimos fazer isso ao fornecer o xpath deste elemento. Com a ferramenta de ["inspecionar" um elemento](https://pt.wikihow.com/Inspecionar-um-Elemento-no-Chrome) no Google Chrome, conseguimos acessar o código fonte HTML de qualquer elemento de uma página. Ao acessarmos o código-fonte do hyperlink "pesquisa avançada", podemos simplemente copiar o xpath e incluí-lo como argumento da função (para maiores informações sobre como ler códigos XML ou HTML, [vejam este tutorial do Leonardo Barone](https://github.com/leobarone/cebrap_lab_raspagem_r/blob/master/tutorials/webscraping_tutorial02.Rmd), ou este [tutorial sobre raspagem de dados em HTML produzido pela Escola de Dados](https://escoladedados.org/tutoriais/xpath-para-raspagem-de-dados-em-html/)).

Para clicar no elemento encontrado, basta usarmos a função `$clickElement()`.

Em nosso caso, precisamos clicar tanto em "Projetos e +", quanto em "Pesquisa Avançada". 

```r
# Selecionar projetos no formulario
projetos <- cliente$findElement(using = "xpath", "//*[@id='quicktabs-tab-pesquisa_de_leis-1']")
projetos$clickElement()

# Clicar para entrar na pesquisa avancada
pesquisa_avancada <- cliente$findElement(using = "xpath", "//*[@id='form_pesquisa_proposicoes']/p/u/strong/a")
pesquisa_avancada$clickElement()
```

Com isso, abrimos a página de pesquisa de proposições. Veja o endereço dela no navegador. Ele se alterou? Sim! O que isso quer dizer? Poderíamos ter simplesmente navegado diretamente para esta página com `$navigate()`:

```r
# Acessar diretamente a pagina de pesquisa de navegacao.
cliente$navigate("https://www.cmbh.mg.gov.br/atividade-legislativa/pesquisar-proposicoes")
```

Vamos agora começar a raspar informações de projetos de lei. O primeiro passo que precisamos fazer é selecionar o "Tipo de proposição" que desejamos listar. Novamente, usamos uma combinação de `$findElement` e `$clickElement` para identificar onde se encontra a opção "Projeto de Lei":

```r
# Abrir a janela de tipos de proposica
tipo_proposicao <- cliente$findElement(using = "xpath", "//*[@id='tipo']/option[10]")
tipo_proposicao$clickElement()
```

Perceba que apesar de derivado de `cliente`, `tipo_proposicao` é um objeto de classe diferente. Conseguimos clicar em elementos (ou fazer outras ações como preencher formulários) de objetos da classe `webElement`, mas não em elementos da classe `remoteDriver`. Em geral, vamos usar o `cliente$` para encontrar elementos numa página, guardar esta informação num objeto de classe `webElement` e explorá-los a partir do nome deste objeto.

```rr
class(rD)              # rsClientServer environment
class(cliente)         # remoteDriver
class(tipo_proposicao) # webElement
```

Para diminuir o número de resultados de busca, vamos filtrar nossa busca a projetos de lei de 2019. Ao combinarmos o `$findElement` e `$clickElement` vemos que o cursor começará a piscar no campo "ano". Em seguida, basta preenchermos o campo com `$sendKeysToElement`. Por fim, basta clicarmos em "Pesquisar".

```r
# Selecionar o campo ano
ano_proposicao <- cliente$findElement(using = "xpath", "//*[@id='ano']")
ano_proposicao$clickElement()

# Preencher o campo ano
ano_proposicao$sendKeysToElement(list("2019"))

# Clicar em pesquisar
pesquisar <- cliente$findElement(using = "xpath", "//*[@id='pesquisar']")
pesquisar$clickElement()
```

Abaixo de "Resultados da pesquisa" estão os resultados de nossa busca. O endereço da página se alterou? Não! Trata-se assim de uma página dinâmica e o `RSelenium` será bastante útil para a coleta de seus dados.

Uma das funcionalidades do RSelenium é mover o mouse na tela. Por exemplo, selecionamos a localização de "Resultados da pesquisa" na página e movemos o mouse até lá. Isso pode ser útil para páginas dinâmicas que dependem de movermos o mouse para abrir novos resultados ([como esta raspagem que fiz do Pocket Casts para o Volt Data Lab](https://github.com/voltdatalab/pesquisa-podcasts/blob/master/raspagem/raspagem_replicacao.md)).

```r
# Selecionar "resultados da pesquisa"
webElems <- cliente$findElement(using = "xpath", "//*[@id='resultadoPesquisaInterno']/h1")

# Mover o mouse até o elemento escolhido
cliente$mouseMoveToLocation(webElement = webElems)
```

Vamos agora raspar algumas informações básicas sobre esses projetos. Inspecionando os nossos resultados de busca, percebemos que é possível rasparmos as seguintes informações:

* Número;
* Ano;
* Autoria;
* Ementa;
* Assunto;
* Situação;
* Fase Atual.

Todas essas informações são elementos textuais da página e podem ser extraídas em blocos referentes a cada um dos PLs. Para tanto, vamos utilizar algumas funções do pacote `stringr` para nos ajudar a identificar cada um desses elementos. 

### Raspando somente o primeiro resultado da pesquisa de projetos de lei

Vamos começar com o exemplo do "Projeto de Lei - 908/2019". Para isso, nós extraímos todos os textos referentes a esse PL com a função `$getElementText()` (veja o resultado desta função sem incluirmos o complemento `[[1]]`. Você entendeu o por que dele ser necessário?), além do "cabeçalho" onde consta o número e o ano do PL. 

```r
# Extraindo o texto de numero e ano
cabecalho <- cliente$findElement(using = "xpath", "//*[@id='resultadoPesquisaInterno']/ul[2]/li[1]/h3/span[1]")
cabecalho <- cabecalho$getElementText()[[1]]

# Extraindo os demais textos
textos <- cliente$findElement(using = "xpath", "//*[@id='resultadoPesquisaInterno']/ul[2]/li[1]/div")
textos <- textos$getElementText()[[1]]
```

Nossos resultados são dois vetores, cada um com um único elemento de texto. Para conseguirmos extrair as informações que desejamos, precisamos identificar expressões regulares que serão os delimitadores da posição dos caracteres a partir da qual vamos extrair as informações. A partir delas podemos selecionar os characters que desejamos extrair de cada um dos vetores. Por fim, basta guardar essas informações num banco de dados.  

```r
# Carregar o pacote stringr
library(stringr)

# Identificar o inicio e fim da coleta de cada elemento
numeros   <- str_locate_all(cabecalho, "Projeto de Lei")[[1]]
anos      <- str_locate_all(cabecalho, "Projeto de Lei")[[1]]+8
autoria   <- str_locate_all(textos,    "Autoria: "     )[[1]]
ementas   <- str_locate_all(textos,    "Ementa: "      )[[1]]
assuntos  <- str_locate_all(textos,    "Assunto: "     )[[1]]
situacoes <- str_locate_all(textos,    "Situação: "    )[[1]]
fases     <- str_locate_all(textos,    "Fase Atual: "  )[[1]]
fim       <- str_locate_all(textos,     "é a favor"     )[[1]]

# Extrair as informacoes a partir dos delimitadores da posicao no character
numero   <- substr(cabecalho,  numeros[1,2]+4,    numeros[1,2]+6)
ano      <- substr(cabecalho,  anos[1,2],         anos[1,2]+3)
autor    <- substr(textos,     autoria[1,2]+1,    ementas[1,1]-2)
ementa   <- substr(textos,     ementas[1,2]+1,    assuntos[1,1]-2)
assunto  <- substr(textos,     assuntos[1,2]+1,   situacoes[1,1]-2)
situacao <- substr(textos,     situacoes[1,2]+1,  fases[1,1]-2)
fase     <- substr(textos,     fases[1,2]+1,      nchar(textos))

# Guardar informacao em data.frame
banco_pl <- data.frame(numero, ano, autor, ementa, assunto, situacao, fase)
```

### Raspando somente a primeira página de resultados da pesquisa de projetos de lei

Contudo, o que nós queremos é automatizar a coleta. Podemos então simplesmente repetir isso para todos os projetos de lei da página. Vamos utilizar `for` loops, colocando pausas (`Sys.sleep`) para evitar que o código quebre (pausas são **muito importantes** no uso do `RSelenium`).

```r
# Criar um objeto chamado banco_final onde guardaremos a informacao de cada PL
banco_final <- NULL

for(i in 1:7){
  
  # Extraindo o texto de numero e ano
  cabecalho <- cliente$findElement(using = "xpath", paste0("//*[@id='resultadoPesquisaInterno']/ul[2]/li[", i, "]/h3/span[1]"))
  cabecalho <- cabecalho$getElementText()[[1]]

  Sys.sleep(1.0)

  # Extraindo os demais textos
  textos <- cliente$findElement(using = "xpath", paste0("//*[@id='resultadoPesquisaInterno']/ul[2]/li[", i, "]/div"))
  textos <- textos$getElementText()[[1]]

  # Identificar o inicio e fim da coleta de cada elemento
  numeros   <- str_locate_all(cabecalho, "Projeto de Lei")[[1]]
  anos      <- str_locate_all(cabecalho, "Projeto de Lei")[[1]]+8
  autoria   <- str_locate_all(textos,    "Autoria: "     )[[1]]
  ementas   <- str_locate_all(textos,    "Ementa: "      )[[1]]
  assuntos  <- str_locate_all(textos,    "Assunto: "     )[[1]]
  situacoes <- str_locate_all(textos,    "Situação: "    )[[1]]
  fases     <- str_locate_all(textos,    "Fase Atual: "  )[[1]]
  fim       <- str_locate_all(textos,     "é a favor"     )[[1]]

  # Extrair as informacoes a partir dos delimitadores da posicao no character
  numero   <- substr(cabecalho,  numeros[1,2]+4,    numeros[1,2]+6)
  ano      <- substr(cabecalho,  anos[1,2],         anos[1,2]+3)
  autor    <- substr(textos,     autoria[1,2]+1,    ementas[1,1]-2)
  ementa   <- substr(textos,     ementas[1,2]+1,    assuntos[1,1]-2)
  assunto  <- substr(textos,     assuntos[1,2]+1,   situacoes[1,1]-2)
  situacao <- substr(textos,     situacoes[1,2]+1,  fases[1,1]-2)
  fase     <- substr(textos,     fases[1,2]+1,      nchar(textos))

  Sys.sleep(1.0)

  # Guardar informacao em data.frame
  banco_pl <- data.frame(numero, ano, autor, ementa, assunto, situacao, fase)

  # Empilhar a informacao do PL atual com todos os PLs anteriores
  banco_final <- rbind.data.frame(banco_final, banco_pl)

  print(i)
  
}
```

Conseguimos raspar todas as informações de uma mesma página! Mas nós queremos raspar informações da busca completa. Como fazemos isso?

### Raspando todos os resultados da pesquisa

Incluímos um novo loop, que raspa as informações de cada página, clica na setinha para avançar de página, coleta a página seguinte e assim sucessivamente. Foi necessário fazer alguns ajustes que são explicados nos comentários do código. Veja se você consegue entender todos.

```r
# Criar um objeto chamado banco_final onde guardaremos a informacao de cada PL
banco_final  <- NULL

# O loop do numero de paginas dependera do numero total de itens, dividido pelo numero de itens em cada pagina (no caso, 215 e 7, respectivamente).
for(j in 1:round(215/7)){
  
  # Printa uma mensagem indicando a pagina que esta sendo coletada
  print(paste("==========  Iniciada  pagina", j, "=========="))
  
  Sys.sleep(0.5)
  
  # Extrai cada PL da pagina
  for(i in 1:7){
    
    # Extraindo o texto de numero e ano
    cabecalho <- cliente$findElement(using = "xpath", paste0("//*[@id='resultadoPesquisaInterno']/ul[2]/li[", i, "]/h3/span[1]"))
    cabecalho <- cabecalho$getElementText()[[1]]
    
    Sys.sleep(1.0)
    
    # Extraindo os demais textos
    textos <- cliente$findElement(using = "xpath", paste0("//*[@id='resultadoPesquisaInterno']/ul[2]/li[", i, "]/div"))
    textos <- textos$getElementText()[[1]]
    
    # Identificar o inicio e fim da coleta de cada elemento
    numeros   <- str_locate_all(cabecalho, "Projeto de Lei")[[1]]
    anos      <- str_locate_all(cabecalho, "Projeto de Lei")[[1]]+8
    autoria   <- str_locate_all(textos,    "Autoria: "     )[[1]]
    ementas   <- str_locate_all(textos,    "Ementa: "      )[[1]]
    assuntos  <- str_locate_all(textos,    "Assunto: "     )[[1]]
    situacoes <- str_locate_all(textos,    "Situação: "    )[[1]]
    fases     <- str_locate_all(textos,    "Fase Atual: "  )[[1]]
    fim       <- str_locate_all(textos,     "é a favor"     )[[1]]

    # Extrair as informacoes a partir dos delimitadores da posicao no character
    numero   <- substr(cabecalho,  numeros[1,2]+4,    numeros[1,2]+6)
    ano      <- substr(cabecalho,  anos[1,2],         anos[1,2]+3)
    autor    <- substr(textos,     autoria[1,2]+1,    ementas[1,1]-2)
    ementa   <- substr(textos,     ementas[1,2]+1,    assuntos[1,1]-2)
    assunto  <- substr(textos,     assuntos[1,2]+1,   situacoes[1,1]-2)
    
    # Como alguns dos PLs ja foram retirados, promulgados em lei ou rejeitados,
    # precisamos fazer um ajuste. Para tanto, colocamos um if que indica que 
    # caso alguma dessas situacoes tenha ocorrido, nao teremos a fase em que 
    # o PL se encontra, e preencheremos de forma diferente a situacao dele.
    
    if(grepl("Retirada|Lei|Rejeitada", textos) == T){
      situacao <- substr(textos,     situacoes[1,2]+1, situacoes[1,2]+9)
      fase = NA
      } 
    
    else 
      
      {
    situacao <- substr(textos,     situacoes[1,2]+1,  fases[1,1]-2)
    fase <- substr(textos,     fases[1,2]+1,      nchar(textos))
    }

    Sys.sleep(1.0)
    
    # Printa uma mensagem indicando o PL cuja informacao esta sendo obtida.
    print(paste0("PL ", numero, "/", ano))

    # Guardar informacao do PL em data.frame
    banco_pl <- data.frame(numero, ano, autor, ementa, assunto, situacao, fase)
    
    # Empilhar a informacao do PL atual com todos os PLs anteriores
    banco_final <- rbind.data.frame(banco_final, banco_pl)
    
  }
  
  if(j == round(215/7)){
  
    print("Fim da coleta.")
      
  } else
  
  # Clica na setinha para mudar de pagina
  prox_pag <- cliente$findElement(using = "xpath", "//*[@id='resultadoPesquisaInterno']/ul[1]/li[8]/a")
  prox_pag$clickElement()
  
  # Printa uma mensagem indicando a pagina cuja coleta foi finalizada
  print(paste("========== Finalizada pagina", j, "=========="))
  
  Sys.sleep(3.0)

}
```

Agora temos um banco completo (chamado `banco_final`), com todos os projetos de lei propostos na CMBH em 2019. Ele ainda precisa de uma limpeza, que vocês podem fazer com o conhecimento de limpeza de banco de dados (especialmente de texto) de tutoriais anteriores (se necessário, [volte aos tutoriais de mineração de textos, produzido pelo Leo Barone e Jonathan Phillips](https://github.com/leobarone/FLS6397_2018/blob/master/classes/class10.md)).

Com nossa coleta de dados encerrada, **não se esqueça de encerrar o cliente e o servidor** Selenium que abrimos ao inicio da sessão.

```r
cliente$close()
rD$server$stop()
```

## Considerações finais e sugestões de leitura

Terminamos assim nossa raspagem de dados com o `RSelenium`. Neste tutorial, apresentamos uma introdução de alguns elementos do pacote: como abrir um servidor Selenium, carregar uma página, selecionar elementos de uma página, clicar, preencher formulários, mover o mouse e raspar algumas informações.

O `RSelenium` possui outras funcionalidades que vocês vão descobrir a medida que continuarem usando-o. Se quiserem conhecer mais, uma leitura obrigatória é a [vinheta de informações básicas do pacote no CRAN](https://cran.r-project.org/web/packages/RSelenium/vignettes/basics.html).

Além disso, os links a seguir apresentam tutoriais e exemplos de outras raspagens com o pacote:

* [Tutorial de uso do RSelenium, apresentação de slides de Daniel Marcelino, 2015](http://danielmarcelino.github.io/pt/talk/2015-06-07-tutorial-r-selenium/)
* [Coletando dados do Google Sheets via RSelenium (em inglês)](https://www.r-bloggers.com/web-scraping-google-sheets-with-rselenium/);
* [Coletando dados de latitude e longitude (em inglês)](https://thatdatatho.com/2019/01/22/tutorial-web-scraping-rselenium/).
