---
layout: post
title:  "amCharts in R"
date:   2015-09-13 20:13:08
categories: Visualization 
---

 
The javascript library [amCharts][amCharts] is an amazing chart library, consists of three parts: Charts, Stock Charts and Maps.
[rAmCharts][rAmCharts] is R API to amCharts.  Fortunally, this package finished percentage is very high, almost including all interface of the amCharts. The js library API is [here][amChartAPI].

The package installation is through github:

```r
devtools::install_github("dataKnowledge/rAmCharts")
```


There're several examples in rAmCharts github project [web][ramchartweb],
Let's see the stock example.

```r
library(rAmCharts)
library(pipeR)

# ---------------------------
# Create the dataProvider ---
# ---------------------------
firstDate <- Sys.Date()
chartData1 <- as.data.frame(t(sapply(0:20, FUN = function(i)
{
  date <- format(firstDate + i, "%m/%d/%Y")
  a <- round(runif(1) * (40 + i)) + 100 + i
  b <- round(runif(1) * (1000 + i)) + 500 + i * 2
  c(date = date, value = a,  volume = b)
})))

chartData2 <- as.data.frame(t(sapply(0:20, FUN = function(i)
{
  date <- format(firstDate + i, "%m/%d/%Y")
  a <- round(runif(1) * (100 + i)) + 200 + i
  b <- round(runif(1) * (1000 + i)) + 600 + i * 2
  c(date = date, value = a,  volume = b)
})))

chartData3 <- as.data.frame(t(sapply(0:20, FUN = function(i)
{
  date <- format(firstDate + i, "%m/%d/%Y")
  a <- round(runif(1) * (100 + i)) + 200 + i
  b <- round(runif(1) * (1000 + i)) + 600 + i * 2
  c(date = date, value = a,  volume = b)
})))

chartData4 <- as.data.frame(t(sapply(0:20, FUN = function(i)
{
  date <- format(firstDate + i, "%m/%d/%Y")
  a <- round(runif(1) * (1s00 + i)) + 200 + i
  b <- round(runif(1) * (1000 + i)) + 600 + i * 2
  c(date = date, value = a,  volume = b)
})))

# --------------
# Draw chart ---
# --------------
amStockChart(theme = "light"
) %>>% addDataSet(dataSet(title = "first data set", categoryField = "date",
                            dataProvider = chartData1) %>>%
                     addFieldMapping(fromField = "value", toField = "value") %>>%
                     addFieldMapping(fromField = "volume", toField = "volume")
) %>>% addDataSet(dataSet(title = "second data set", categoryField = "date",
                            dataProvider = chartData2) %>>%
                     addFieldMapping(fromField = "value", toField = "value") %>>%
                     addFieldMapping(fromField = "volume", toField = "volume")
) %>>% addDataSet(dataSet(title = "third data set", categoryField = "date",
                            dataProvider = chartData3) %>>%
                     addFieldMapping(fromField = "value", toField = "value") %>>%
                     addFieldMapping(fromField = "volume", toField = "volume")
) %>>% addDataSet(dataSet(title = "fourth data set", categoryField = "date",
                            dataProvider = chartData4) %>>%
                     addFieldMapping(fromField = "value", toField = "value") %>>%
                     addFieldMapping(fromField = "volume", toField = "volume")
) %>>% addPanel(stockPanel(showCategoryAxis = FALSE, title = "Value", percentHeight = 70) %>>%
                   addStockGraph(id = "g1", valueField = "value", comparable = TRUE,
                                 compareField = "value", balloonText = "[[title]] =<b>[[value]]</b>",
                                 compareGraphBalloonText = "[[title]] =<b>[[value]]</b>"
                  ) %>>% setStockLegend(periodValueTextComparing = "[[percents.value.close]]%",
                                          periodValueTextRegular = "[[value.close]]")
) %>>% addPanel(stockPanel(title = "Volume", percentHeight = 30) %>>%
                   addStockGraph(valueField = "volume", type = "column",
                                 fillAlphas = 1) %>>%
                   setStockLegend(periodValueTextRegular = "[[value.close]]")
) %>>% setChartScrollbarSettings(graph = "g1"
) %>>% setChartCursorSettings(valueBalloonsEnabled = TRUE, fullWidth = TRUE,
                               cursorAlpha = 0.1, valueLineBalloonEnabled = TRUE,
                               valueLineEnabled = TRUE, valueLineAlpha = 0.5
) %>>% setPeriodSelector(periodSelector(position = "left") %>>%
                            addPeriod(period = "MM", selected = TRUE, count = 1, label = "1 month") %>>%
                            addPeriod(period = "MAX", label = "MAX")
) %>>% setDataSetSelector(position = "left"
) %>>% setPanelsSettings(recalculateToPercents = FALSE
) %>>% plot
```

It also could be used in shiny, see the `shinyExample()` in that package. Guess the points are 

1. use `rAmCharts::renderAmCharts` wrap the chart
2. use `setExport()`
3. use `rAmCharts::amChartsOutput` in GUI.

```r
#' @title Few shiny examples
#' @description Launch a shiny app with amChart examples
#' @examples
#' \donttest{
#' library(pipeR)
#' library(shiny)
#' shinyExamples()
#' }
#' @export
shinyExamples <- function()
{
  # Check package dependencies
  stopifnot(require(shiny))
  stopifnot(require(pipeR))
  
  # Load App
  shinyApp(
    ui <- fluidPage(
      # Application title
      titlePanel("Charts by rAmCharts"),
      
      fluidRow(
        column(
          width = 6,
          rAmCharts::amChartsOutput("radar", type = "radar"),
          rAmCharts::amChartsOutput("radar2", type = "radar"),
          rAmCharts::amChartsOutput("pie", type = "pie")
        ),
        column(
          width = 6,
          rAmCharts::amChartsOutput("serial", type = "serial"),
          rAmCharts::amChartsOutput("drillColumnChart1", type = "drill"),
          rAmCharts::amChartsOutput("drillColumnChart2", type = "drill")
        )
      )
    ),
    server <- function(input, output) {
      
      category <- reactive({
        "attribute"
      })
      
      output$radar <- rAmCharts::renderAmCharts({
        amRadarChart( 
          startDuration = 1, categoryField = category(), theme = "dark"
        ) %>>% setDataProvider(
          data.frame(
            attribute = c("data", "brand", "singleness"), p1 = c(.3, -1, 0), p2 = c(.7, 1, 2)
          )
        ) %>>% addGraph( balloonText = "Utility : [[value]]", valueField = "p1", title = "p1"
        ) %>>% addGraph( balloonText = "Utility : [[value]]", valueField = "p2", title = "p2"
        ) %>>% setLegend( useGraphSettings = TRUE
        ) %>>% setExport() %>>% plot  
      })
      
      output$radar2 <- rAmCharts::renderAmCharts({
        amRadarChart( 
          startDuration = 1, categoryField = "attribute", theme = "chalk", creditsPosition = "bottom-right"
        ) %>>% setDataProvider(
          data.frame(
            attribute = c("data", "brand", "singleness"), p1 = c(.3, -1, 0), p2 = c(.7, 1, 2)
          )
        ) %>>% addGraph( balloonText = "Utility : [[value]]", valueField = "p1", title = "p1"
        ) %>>% addGraph( balloonText = "Utility : [[value]]", valueField = "p2", title = "p2"
        ) %>>% setLegend( useGraphSettings = TRUE
        ) %>>% setExport() %>>% plot  
      })
      
      output$pie <- rAmCharts::renderAmCharts({
        amPieChart(valueField = "value", titleField = "key", creditsPostion = "top-right"
        ) %>>% setDataProvider(data.frame(key = c("FR", "US"), value = c(20,10))
        ) %>>% setExport() %>>% plot
      })
      
      output$serial <- rAmCharts::renderAmCharts({
        amSerialChart( categoryField = "country", creditsPosition = "top-right", theme = "light"
        ) %>>% setDataProvider(data.frame(country = c("FR", "US"), visits = 1:2)
        ) %>>% addGraph( balloonText = "[[category]]: <b>[[value]]</b>", type = "column",
                         valueField = "visits", fillAlphas = 0.8, lineAlpha = 0.2
        ) %>>% setExport() %>>% plot
      })
      
      output$drillColumnChart1 <-rAmCharts::renderAmCharts({
        df <- data.frame(
          name = c("data", "Brand", "singleness"), start = c(8,10,6), end = c(11,13,10),
          color = c('#007FFF', "#007FFF", "#003FFF"), description = c("click to drill-down","","")
        )
        
        amSerialChart( categoryField = "name", dataProvider = df
        ) %>>% addGraph(valueField = "end", type = "column", openField = "start", lineAlpha = 0,
                        fillColorsField = "color", fillAlphas = 0.9,
                        balloonText = "<b>[[category]]</b><br>from [[start]] to [[end]]<br>[[description]]"
        ) %>>% addSubData( 1, data.frame( modality = c("3G", "4G"), utility = c(-1,2), 
                                          color = c("#007FFF", "#007FFF") )
        ) %>>% setSubChartProperties(.subObject = amSerialChart(creditsPosition = "bottom-right", categoryField = "modality"
        ) %>>% addGraph(valueField = 'utility', type = 'column', categoryField = "modality",
                        lineAlpha = 0, fillColorsField = "color", fillAlphas = 0.9,
                        balloonText = "[[modality]]: <b>[[utility]]</b>")
        ) %>>% setExport() %>>% plot
      })
      
      output$drillColumnChart2 <-rAmCharts::renderAmCharts({
        amSerialChart(categoryField = "name", theme = "chalk"
        ) %>>%setDataProvider( data.frame( name = c("data", "Brand", "singleness"),
                                           start = c(8,10,6), end = c(11,13,10),
                                           color = c('#007FFF', "#007FFF", "#003FFF"),
                                           description = c("click to drill-down","","") )
        ) %>>% addGraph( valueField = "end", type = "column", openField = "start", lineAlpha = 0,
                         fillColorsField = "color", fillAlphas = 0.9,
                         balloonText = "<b>[[category]]</b><br>from [[start]] to [[end]]<br>[[description]]"
        ) %>>% addSubData( 1, data.frame( modality = c("3G", "4G"), utility = c(-1,2), 
                                          color = c("#007FFF", "#007FFF") )
        ) %>>% setSubChartProperties(.subObject = amSerialChart(creditsPosition = "bottom-right", categoryField = "modality"
        ) %>>% addGraph( valueField = 'utility', type = 'column', categoryField = "modality",
                         lineAlpha = 0, fillColorsField = "color", fillAlphas = 0.9,
                         balloonText = "[[modality]]: <b>[[utility]]</b>" )
        ) %>>% setExport() %>>% plot
      })
    }
  )
}
```

btw: The logo will show in the cornor of the chart, we can comment a paragraph of js to [avoid it][avoid].

[amCharts]: http://www.amcharts.com/
[rAmCharts]: https://github.com/DataKnowledge/rAmCharts
[ramchartweb]: http://dataknowledge.github.io/rAmCharts/
[amChartAPI]: http://docs.amcharts.com/3/
[avoid]: http://blog.csdn.net/aoxida/article/details/8109335


