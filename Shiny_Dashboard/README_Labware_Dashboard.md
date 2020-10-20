****Library and source****

``` r
library(shiny)
library(DBI)
library(shinydashboard)
```

****Shiny**** SQL is based on the analysis report. The sample must have
a station linked to it to ensure that only projects that are available
in AquaMonitor are available in the dashboard. Username and password are
stored in the Renviron.

``` r
ui <- dashboardPage(
  dashboardHeader(title = "Labware dashboard"),
  dashboardSidebar( width = 150,
    downloadButton('downloadData',
                   'Download',
                   style = "color: #fff; background-color: #27ae60; border-color: #00a9e0")
  ),
  dashboardBody(fluidRow(
    box(
      p("Velg LIMS oppdrag"),
      textInput("PROJECT1", "Oppdragsnr:", "")),
    
    tableOutput("test")
  
  ))
)

server <- function(input, output, session) {
  output$test <- renderTable({
    conn <-
      DBI::dbConnect(
        odbc::odbc(),
        'LW_Prod',
        UID = Sys.getenv('LABWAREUSER'),
        PWD = Sys.getenv('LABWAREPASS'),
        encoding = "windows-1252"
      )
    
    
sql <- 
"SELECT 
  SAMPLE.TEXT_ID as Prøvenr,
  SAMPLE.DESCRIPTION as Prøvemerking,
  V_NIVA_PROJ_STATIONS.STATION_NAME as Stasjon,
  RESULT.NAME as analyse,
  V_NIVA_ANA_FRACTIONS.ANA_FRACTION as Fraksjon,
  TEST.X_ISO_REF as Standard,
  RESULT.FORMATTED_ENTRY as Resultat,
  UNITS.DISPLAY_STRING as enheter,
  RESULT.ACCREDITED as Akkreditering_status,
  VENDOR.X_VENDOR_ACCID as Akkreditering_Underleverandør,
  VENDOR.DESCRIPTION as Underleverandør,
  SAMPLE.PROJECT as Prosjekt,
  TEST.T_ANALYSIS_METHOD as Metode_Kode,
  SAMPLE.SAMPLE_TYPE as Prøvetype,
  V_NIVA_SPECIES.NORWEGIAN_NAME as Art,
  V_NIVA_TISSUE_TYPES.TISSUE_NAME as Vev 
FROM
((((((((((((
  (LABWARE.PROJECT PROJECT 
    INNER JOIN LABWARE.SAMPLE SAMPLE ON PROJECT.NAME=SAMPLE.PROJECT) 
  LEFT OUTER JOIN LABWARE.LIMS_NOTES LIMS_NOTES_PROJ 
    ON PROJECT.X_COMMENT=LIMS_NOTES_PROJ.NOTE_ID) 
  INNER JOIN LABWARE.CUSTOMER CUSTOMER
    ON PROJECT.CUSTOMER=CUSTOMER.NAME) 
  INNER JOIN LABWARE.ACCOUNT ACCOUNT 
    ON PROJECT.X_ACCOUNT=ACCOUNT.ACCOUNT_NUMBER)  
  LEFT OUTER JOIN LABWARE.TEST 
    TEST ON SAMPLE.SAMPLE_NUMBER=TEST.SAMPLE_NUMBER) 
  LEFT OUTER JOIN LABWARE.LIMS_NOTES
    LIMS_NOTES_SAMP ON SAMPLE.X_COMMENT_NOTE=LIMS_NOTES_SAMP.NOTE_ID)
  LEFT OUTER JOIN LABWARE.V_NIVA_PROJ_STATIONS V_NIVA_PROJ_STATIONS 
    ON SAMPLE.X_STATION_ID=V_NIVA_PROJ_STATIONS.PROJECTS_STATION_ID) 
  LEFT OUTER JOIN LABWARE.V_NIVA_TISSUE_TYPES V_NIVA_TISSUE_TYPES 
    ON SAMPLE.X_TISSUE_ID=V_NIVA_TISSUE_TYPES.TISSUE_ID) 
  LEFT OUTER JOIN LABWARE.V_NIVA_SED_SAMP_METH 
    V_NIVA_SED_SAMP_METH ON SAMPLE.X_SMPL_METHOD=V_NIVA_SED_SAMP_METH.METHOD_ID) 
  LEFT OUTER JOIN LABWARE.V_NIVA_SPECIES V_NIVA_SPECIES 
    ON SAMPLE.X_SPECIES_ID=V_NIVA_SPECIES.SPECIE_ID)
  LEFT OUTER JOIN LABWARE.RESULT RESULT ON (TEST.SAMPLE_NUMBER=RESULT.SAMPLE_NUMBER) 
  AND V_NIVA_PROJ_STATIONS.STATION_NAME is not null
  AND (TEST.TEST_NUMBER=RESULT.TEST_NUMBER)) 
    LEFT OUTER JOIN LABWARE.V_NIVA_ANA_FRACTIONS V_NIVA_ANA_FRACTIONS
    ON TEST.X_FRACTION_ID=V_NIVA_ANA_FRACTIONS.ANA_FRACTION_ID) LEFT OUTER JOIN LABWARE.UNITS UNITS ON RESULT.UNITS=UNITS.UNIT_CODE) 
  LEFT OUTER JOIN LABWARE.VENDOR VENDOR ON TEST.X_VENDOR_NAME=VENDOR.NAME WHERE (RESULT.reportable = 'T')  
  AND 
    SAMPLE.PROJECT = (?PROJECT1)
  
    
ORDER BY 
  SAMPLE.TEXT_ID, TEST.ANALYSIS, RESULT.NAME "

   
     query <- sqlInterpolate(conn, sql, PROJECT1 = input$PROJECT1) #, PROJECT2 = input$PROJECT2)
   data <- dbGetQuery(conn,query)
        # Download output of check's ----
    output$downloadData <-   
     downloadHandler(
      filename = function () {
      paste( input$PROJECT1,"_", Sys.Date(), ".csv", sep = "")},
      content = function(file) {
        write.csv(data, file)
      } )                          
  dbGetQuery(conn,query)
    
  })
}

shinyApp(ui, server)
```
