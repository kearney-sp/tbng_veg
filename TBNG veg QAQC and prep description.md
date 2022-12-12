# TBNG vegetation QAQC and prep

#### PVSAGE
* Dates
  * Manually changed non-'date-like' values to dates. If date values were a range, took the midrange of the date range.
* Biomass clipping
  * Removed all rows with NA's and negative values (signifying NA) for total biomass
  * Removed all rows with negative values in functional group biomass
  * Filled all empty/nA values in functional group biomass with 0's for averaging
  * Added Brome and VUOC weights together to create 'Annual Grass Weight' field
  * Created a consistent 'Forb Weight' field across all years by adding AF and PF when present
  * Created a consistent 'C4 weight' across years by adding BOBU and other C4 when present
  * Recalculated areal density of all FG weights by dividing by area

* Pin frame

  