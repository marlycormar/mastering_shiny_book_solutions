# Uploads and Downloads

### Exercise 9.4.1 {-}

Use the [ambient](https://ambient.data-imaginist.com) package by Thomas Lin
Pedersen to generate [worley noise](https://ambient.data-imaginist.com/reference/noise_worley.html)
and download a PNG of it.

:::solution
#### Solution {-}

A general method for saving a png file is to select the png driver using the
function `png()`. The only argument the driver needs is a filename (this will
be stored relative to your current working directory!). You will not see the
plot when running the `plot` function because it is being saved to that file
instead. When we're done plotting, we used the `dev.off()` command to close the
connection to the driver.


```r
library(ambient)
noise <- ambient::noise_worley(c(100, 100))

png("noise_plot.png")
plot(as.raster(normalise(noise)))
dev.off()
```
:::

<!---------------------------------------------------------------------------->
<!---------------------------------------------------------------------------->
<!---------------------------------------------------------------------------->

### Exercise 9.4.2 {-}

Create an app that lets you upload a csv file, select a variable, and then
perform a `t.test()` on that variable. After the user has uploaded the csv
file, you'll need to use `updateSelectInput()` to fill in the available
variables. See Section
[10.1](https://mastering-shiny.org/action-dynamic.html#updating-inputs)
for details.

:::solution
#### Solution {-}

We can use the `fileInput` widget with the `accept` argument set to `.csv` to
allow only the upload of csv files. In the `server` function we save the
uploaded data to the the `data` reactive and use it to update `input$variable`,
which displays variable (i.e. numeric data column) choices. Note that we put
the `updateSelectInput` within an observe event because we need the
`input$variable` to change if the user selects another file.


```r
library(shiny)

ui <- fluidPage(
  sidebarLayout(
    sidebarPanel(
      fileInput("file", "Upload CSV", accept = ".csv"), # file widget
      selectInput("variable", "Select Variable", choices = NULL) # select widget
    ),
    mainPanel(
      verbatimTextOutput("results") # t-test results
    )
  )
)

server <- function(input, output,session) {
  
  # get data from file
  data <- reactive({
    req(input$file)
    
    # as shown in the book, lets make sure the uploaded file is a csv
    ext <- tools::file_ext(input$file$name)
    validate(need(ext == "csv", "Invalid file. Please upload a .csv file"))
    
    dataset <- vroom::vroom(input$file$datapath, delim = ",")
    
    # let the user know if the data contains no numeric column
    validate(need(ncol(dplyr::select_if(dataset, is.numeric)) != 0,
                  "This dataset has no numeric columns."))
    dataset
  })
  
  # create the select input based on the numeric columns in the dataframe
  observeEvent(input$file, {
    req(data())
    num_cols <- dplyr::select_if(data(), is.numeric)
    updateSelectInput(session, "variable", choices = colnames(num_cols))
  })
  
  # print t-test results
  output$results <- renderPrint({
    if(!is.null(input$variable))
      t.test(data()[input$variable])
  })
}

shinyApp(ui, server)
```
:::

<!---------------------------------------------------------------------------->
<!---------------------------------------------------------------------------->
<!---------------------------------------------------------------------------->

### Exercise 9.4.3 {-}

Create an app that lets the user upload a csv file, select one variable,
draw a histogram, and then download the histogram. For an additional challenge,
allow the user to select from .png, .pdf, and .svg output formats.

:::solution
#### Solution {-}

Adapting the code from the example above, rather than print a t-test output, we
save the plot in a reactive and use it to display the plot/download. We can use
the `ggsave` function to switch between `input$extension` types.


```r
library(shiny)
library(ggplot2)

ui <- fluidPage(
  tagList(
    br(), br(),
    column(4,
           wellPanel(
             fileInput("file", "Upload CSV", accept = ".csv"),
             selectInput("variable", "Select Variable", choices = NULL),
           ),
           wellPanel(
             radioButtons("extension", "Save As:",
                          choices = c("png", "pdf", "svg"), inline = TRUE),
             downloadButton("download", "Save Plot")
           )
    ),
    column(8, plotOutput("results"))
  )
)

server <- function(input, output,session) {
  
  # get data from file
  data <- reactive({
    req(input$file)
    
    # as shown in the book, lets make sure the uploaded file is a csv
    ext <- tools::file_ext(input$file$name)
    validate(need(ext == "csv", "Invalid file. Please upload a .csv file"))
    
    dataset <- vroom::vroom(input$file$datapath, delim = ",")
    
    # let the user know if the data contains no numeric column
    validate(need(ncol(dplyr::select_if(dataset, is.numeric)) != 0,
                  "This dataset has no numeric columns."))
    dataset
  })
  
  # create the select input based on the numeric columns in the dataframe
  observeEvent( input$file, {
    req(data())
    num_cols <- dplyr::select_if(data(), is.numeric)
    updateSelectInput(session, "variable", choices = colnames(num_cols))
  })
  
  # plot histogram
  plot_output <- reactive({
    req(!is.null(input$variable))
    
    ggplot(data()) +
      aes_string(x = input$variable) +
      geom_histogram()
  })
  
  output$results <- renderPlot(plot_output())
  
  # save histogram using downloadHandler and plot output type
  output$download <- downloadHandler(
    filename = function() {
      paste("histogram", input$extension, sep = ".")
    },
    content = function(file){
      ggsave(file, plot_output(), device = input$extension)
    }
  )
}

shinyApp(ui, server)
```
:::

<!---------------------------------------------------------------------------->
<!---------------------------------------------------------------------------->
<!---------------------------------------------------------------------------->

### Exercise 9.4.4 {-}

Write an app that allows the user to create a Lego mosaic from any .png file
using Ryan Timpe's [brickr](https://github.com/ryantimpe/brickr) package. Once
you've completed the basics, add controls to allow the user to select the size
of the mosaic (in bricks), and choose whether to use "universal" or "generic"
colour palettes.

:::solution
#### Solution {-}

Instead of limiting our file selection to a csv as above, here we are going to
limit our input to a png. We'll use the `png::readPNG` function to read in our
file, and specify the size/color of our mosaic in `brickr`'s `image_to_mosaic`
function. Read more about the package and examples
[here](https://github.com/ryantimpe/brickr).


```r
library(shiny)
library(brickr)
library(png)

# Function to provide user feedback (checkout Chapter 8 for more info).
notify <- function(msg, id = NULL) {
  showNotification(msg, id = id, duration = NULL, closeButton = FALSE)
}

ui <- fluidPage(
  sidebarLayout(
    sidebarPanel(
      fluidRow(
        fileInput("myFile", "Upload a PNG file", accept = c('image/png')),
        sliderInput("size", "Select size:", min = 1, max = 100, value = 35),
        radioButtons("color", "Select color palette:", choices = c("universal", "generic"))
      )
    ),
    mainPanel(
      plotOutput("result"))
  )
)

server <- function(input, output) {

  imageFile <- reactive({
    if(!is.null(input$myFile))
      png::readPNG(input$myFile$datapath)
  })

  output$result <- renderPlot({
    req(imageFile())

    id <- notify("Transforming image...")
    on.exit(removeNotification(id), add = TRUE)

    imageFile() %>%
      image_to_mosaic(img_size = input$size, color_palette = input$color) %>%
      build_mosaic()
  })
}

shinyApp(ui, server)
```
:::
