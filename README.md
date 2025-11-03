# datamiodelleadawareness
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Georgia Lead Risk Map — 159 counties</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <style>
    html,body,#map { height:100%; margin:0; padding:0; }
    #map { position: absolute; inset:0; }
    #sidebar {
      position: absolute;
      left: 12px;
      top: 12px;
      z-index: 1000;
      width: 340px;
      max-height: calc(100% - 24px);
      overflow: auto;
      background: rgba(255,255,255,0.98);
      border-radius: 8px;
      box-shadow: 0 6px 18px rgba(0,0,0,0.15);
      padding: 12px;
      font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
    }
    .county-row { padding:8px; border-bottom:1px solid #eee; cursor:pointer; display:flex; gap:8px; align-items:center }
    .county-row:hover { background:#f3f4f6; }
    .search { width:100%; padding:8px; margin-bottom:8px; border-radius:6px; border:1px solid #ddd; box-sizing:border-box;}
    .legend { margin-top:8px; display:flex; gap:8px; align-items:center; font-size:13px; flex-wrap:wrap }
    .legend .box { width:18px; height:12px; display:inline-block; border-radius:3px; margin-right:6px; border:1px solid #ccc }
    .low { background:#22c55e } .medium { background:#f59e0b } .high { background:#ef4444 }
    .small-muted { font-size:12px;color:#666;margin-top:6px }
    .count-name { font-weight:600 }
    .score-badge { font-weight:700; font-size:16px; }
  </style>
</head>
<body>
  <div id="map"></div>

  <div id="sidebar">
    <input id="search" class="search" placeholder="Search county..." />
    <div id="list" aria-live="polite"></div>
    <div class="legend">
      <div><span class="box low"></span>Low (0–3)</div>
      <div><span class="box medium"></span>Medium (4–6)</div>
      <div><span class="box high"></span>High (7–10)</div>
    </div>
    <div class="small-muted">Click a county in the list or a map dot to view full statistics.</div>
  </div>

  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script src="https://unpkg.com/@turf/turf@6.5.0/turf.min.js"></script>

<script>
/*
  Georgia Lead Risk Map
  - Uses Plotly counties GeoJSON (public) filtered to state FIPS 13 (Georgia)
  - Embedded dataset: the full table you pasted (159 counties)
  - Matches by normalized county name (trims "County", case-insensitive)
*/

const statsData = [
  {"county":"Appling","totalPopulation":"18,497","under20":"4,946","medianIncome":"$51,217","pre1980Units":"3541","elevatedPct":"6.64","riskScore":7,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Atkinson","totalPopulation":"8,559","under20":"2,282","medianIncome":"$46,060","pre1980Units":"1414","elevatedPct":"5.93","riskScore":6,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Bacon","totalPopulation":"11,026","under20":"3,053","medianIncome":"$46,044","pre1980Units":"2364","elevatedPct":"4.55","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Baker","totalPopulation":"2,645","under20":"558","medianIncome":"$50,129","pre1980Units":"855","elevatedPct":"3.12","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Baldwin","totalPopulation":"42,922","under20":"10,684","medianIncome":"$49,883","pre1980Units":"7880","elevatedPct":"1.55","riskScore":2,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Banks","totalPopulation":"20,683","under20":"4,605","medianIncome":"$66,928","pre1980Units":"2252","elevatedPct":"3.39","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Barrow","totalPopulation":"99,886","under20":"25,160","medianIncome":"$77,583","pre1980Units":"6165","elevatedPct":"2.99","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Bartow","totalPopulation":"119,493","under20":"29,044","medianIncome":"$80,017","pre1980Units":"10945","elevatedPct":"2.56","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Ben Hill","totalPopulation":"17,270","under20":"4,575","medianIncome":"$41,141","pre1980Units":"3982","elevatedPct":"3.73","riskScore":4,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Berrien","totalPopulation":"19,280","under20":"4,738","medianIncome":"$48,725","pre1980Units":"3297","elevatedPct":"5.31","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Bibb","totalPopulation":"157,284","under20":"42,251","medianIncome":"$48,539","pre1980Units":"40751","elevatedPct":"3.59","riskScore":4,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Bleckley","totalPopulation":"12,837","under20":"3,112","medianIncome":"$56,070","pre1980Units":"2564","elevatedPct":"9.09","riskScore":9,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Brantley","totalPopulation":"18,783","under20":"4,632","medianIncome":"$50,370","pre1980Units":"2284","elevatedPct":"2.87","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Brooks","totalPopulation":"16,167","under20":"3,676","medianIncome":"$43,353","pre1980Units":"3276","elevatedPct":"3.57","riskScore":4,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Bryan","totalPopulation":"52,781","under20":"15,339","medianIncome":"$93,057","pre1980Units":"2440","elevatedPct":"4.48","riskScore":4,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Bulloch","totalPopulation":"86,905","under20":"23,208","medianIncome":"$49,471","pre1980Units":"8810","elevatedPct":"1.39","riskScore":1,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Burke","totalPopulation":"24,446","under20":"6,492","medianIncome":"$49,431","pre1980Units":"3968","elevatedPct":"4.91","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Butts","totalPopulation":"27,171","under20":"6,104","medianIncome":"$62,231","pre1980Units":"3060","elevatedPct":"11.64","riskScore":12,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Calhoun","totalPopulation":"5,423","under20":"921","medianIncome":"$39,681","pre1980Units":"1256","elevatedPct":"6.58","riskScore":7,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Camden","totalPopulation":"60,412","under20":"15,290","medianIncome":"$67,628","pre1980Units":"3498","elevatedPct":"1.42","riskScore":1,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Candler","totalPopulation":"11,253","under20":"3,195","medianIncome":"$44,493","pre1980Units":"1771","elevatedPct":"2.66","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Carroll","totalPopulation":"132,234","under20":"34,592","medianIncome":"$69,924","pre1980Units":"15421","elevatedPct":"4.25","riskScore":4,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Catoosa","totalPopulation":"69,058","under20":"16373","medianIncome":"$66,918","pre1980Units":"10841","elevatedPct":"4.82","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Charlton","totalPopulation":"13,226","under20":"2,513","medianIncome":"$48,147","pre1980Units":"1559","elevatedPct":"4.49","riskScore":4,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Chatham","totalPopulation":"308,915","under20":"71,154","medianIncome":"$64,029","pre1980Units":"57793","elevatedPct":"3.01","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Chattahoochee","totalPopulation":"8,491","under20":"2,452","medianIncome":"$57,507","pre1980Units":"2037","elevatedPct":"0","riskScore":0,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Chattooga","totalPopulation":"25,742","under20":"6002","medianIncome":"$50,714","pre1980Units":"6347","elevatedPct":"12.12","riskScore":10,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Cherokee","totalPopulation":"297,418","under20":"70,642","medianIncome":"$99,932","pre1980Units":"13343","elevatedPct":"1.94","riskScore":2,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Clarke","totalPopulation":"130,347","under20":"31,555","medianIncome":"49,832","pre1980Units":"23355","elevatedPct":"2.9","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Clay","totalPopulation":"2,853","under20":"626","medianIncome":"$37,961","pre1980Units":"729","elevatedPct":"3.85","riskScore":4,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Clayton","totalPopulation":"298,102","under20":"86,675","medianIncome":"57,625","pre1980Units":"42876","elevatedPct":"2.77","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Clinch","totalPopulation":"6,880","under20":"1,830","medianIncome":"$45,595","pre1980Units":"1357","elevatedPct":"9.09","riskScore":9,"riskLevel":"High","additionalData":"-"},
  {"county":"Cobb","totalPopulation":"786,549","under20":"193,408","medianIncome":"96,798","pre1980Units":"88644","elevatedPct":"3.46","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Coffee","totalPopulation":"43,457","under20":"11,783","medianIncome":"$48,792","pre1980Units":"6099","elevatedPct":"4.96","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Colquitt","totalPopulation":"46,841","under20":"12,947","medianIncome":"$49,214","pre1980Units":"9198","elevatedPct":"4.37","riskScore":4,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Columbia","totalPopulation":"170,544","under20":"44,376","medianIncome":"$94,224","pre1980Units":"9944","elevatedPct":"5.21","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Cook","totalPopulation":"18,324","under20":"4893","medianIncome":"$44,476","pre1980Units":"3356","elevatedPct":"5.29","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Coweta","totalPopulation":"161,918","under20":"38953","medianIncome":"$87,618","pre1980Units":"11033","elevatedPct":"10.15","riskScore":10,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Crawford","totalPopulation":"12,493","under20":"2706","medianIncome":"$60,295","pre1980Units":"1642","elevatedPct":"1.77","riskScore":2,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Crisp","totalPopulation":"19,435","under20":"5057","medianIncome":"$43,884","pre1980Units":"5000","elevatedPct":"3.47","riskScore":3,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Dade","totalPopulation":"16,315","under20":"3621","medianIncome":"$59,708","pre1980Units":"3011","elevatedPct":"8.96","riskScore":9,"riskLevel":"High","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Dawson","totalPopulation":"34,818","under20":"7005","medianIncome":"$84,551","pre1980Units":"1488","elevatedPct":"2.98","riskScore":3,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Decatur","totalPopulation":"29,223","under20":"7731","medianIncome":"$46,455","pre1980Units":"5731","elevatedPct":"7.44","riskScore":7,"riskLevel":"High","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"DeKalb","totalPopulation":"765,418","under20":"189265","medianIncome":"76,736","pre1980Units":"151281","elevatedPct":"3.71","riskScore":4,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Dodge","totalPopulation":"19,582","under20":"4128","medianIncome":"$43,783","pre1980Units":"4292","elevatedPct":"27.14","riskScore":10,"riskLevel":"High","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Dooly","totalPopulation":"11,773","under20":"1896","medianIncome":"$46,191","pre1980Units":"2317","elevatedPct":"6.45","riskScore":6,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Dougherty","totalPopulation":"82,145","under20":"22858","medianIncome":"$42,629","pre1980Units":"23987","elevatedPct":"4.02","riskScore":4,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Douglas","totalPopulation":"152,574","under20":"41413","medianIncome":"$79,242","pre1980Units":"14109","elevatedPct":"1.6","riskScore":2,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Early","totalPopulation":"10,569","under20":"2821","medianIncome":"$43,417","pre1980Units":"2753","elevatedPct":"3.64","riskScore":4,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Echols","totalPopulation":"3,755","under20":"1064","medianIncome":"$47,195","pre1980Units":"580","elevatedPct":"0","riskScore":0,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Effingham","totalPopulation":"76,555","under20":"20407","medianIncome":"$88,111","pre1980Units":"4437","elevatedPct":"1.83","riskScore":2,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Elbert","totalPopulation":"20,293","under20":"4818","medianIncome":"$49,220","pre1980Units":"5043","elevatedPct":"4.44","riskScore":4,"riskLevel":"Medium","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Emanuel","totalPopulation":"23,465","under20":"6209","medianIncome":"$46,975","pre1980Units":"5210","elevatedPct":"3.61","riskScore":4,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Evans","totalPopulation":"10,944","under20":"2909","medianIncome":"$45,163","pre1980Units":"1952","elevatedPct":"1.45","riskScore":1,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Fannin","totalPopulation":"26,399","under20":"4426","medianIncome":"$60,604","pre1980Units":"4365","elevatedPct":"3.01","riskScore":3,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Fayette","totalPopulation":"126,091","under20":"30529","medianIncome":"$109,358","pre1980Units":"7609","elevatedPct":"1.81","riskScore":2,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Floyd","totalPopulation":"101,465","under20":"26206","medianIncome":"$56,651","pre1980Units":"22975","elevatedPct":"6.57","riskScore":7,"riskLevel":"Medium","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Forsyth","totalPopulation":"284,037","under20":"75753","medianIncome":"130,909","pre1980Units":"6780","elevatedPct":"2.9","riskScore":3,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Franklin","totalPopulation":"26,054","under20":"6,206","medianIncome":"$49,940","pre1980Units":"4108","elevatedPct":"3.98","riskScore":4,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Fulton","totalPopulation":"1,089,919","under20":"252192","medianIncome":"89,798","pre1980Units":"190775","elevatedPct":"2.96","riskScore":3,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Gilmer","totalPopulation":"33,774","under20":"6479","medianIncome":"$68,294","pre1980Units":"3348","elevatedPct":"1.19","riskScore":1,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Glascock","totalPopulation":"3,006","under20":"696","medianIncome":"$55,539","pre1980Units":"598","elevatedPct":"6.25","riskScore":6,"riskLevel":"Medium","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Glynn","totalPopulation":"88,226","under20":"19509","medianIncome":"$63,532","pre1980Units":"15422","elevatedPct":"3.49","riskScore":3,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Gordon","totalPopulation":"61,561","under20":"14994","medianIncome":"$63,946","pre1980Units":"7684","elevatedPct":"2.41","riskScore":2,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Grady","totalPopulation":"26,210","under20":"7021","medianIncome":"$48,548","pre1980Units":"4777","elevatedPct":"5.43","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Greene","totalPopulation":"21,944","under20":"3812","medianIncome":"$74,488","pre1980Units":"2846","elevatedPct":"7.4","riskScore":7,"riskLevel":"High","additionalData":"-"},
  {"county":"Gwinnett","totalPopulation":"998,232","under20":"279611","medianIncome":"83,918","pre1980Units":"48257","elevatedPct":"3.44","riskScore":3,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Habersham","totalPopulation":"51,341","under20":"12019","medianIncome":"$62,428","pre1980Units":"6472","elevatedPct":"4.49","riskScore":4,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Hall","totalPopulation":"226,559","under20":"56294","medianIncome":"$76,789","pre1980Units":"20292","elevatedPct":"2.78","riskScore":3,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Hancock","totalPopulation":"9,194","under20":"1409","medianIncome":"$38,570","pre1980Units":"1988","elevatedPct":"9","riskScore":9,"riskLevel":"High","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Haralson","totalPopulation":"33,404","under20":"8440","medianIncome":"$57,912","pre1980Units":"5638","elevatedPct":"6.16","riskScore":6,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Harris","totalPopulation":"37,448","under20":"8576","medianIncome":"$90,636","pre1980Units":"3652","elevatedPct":"5.08","riskScore":5,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Hart","totalPopulation":"28,896","under20":"6061","medianIncome":"$56,137","pre1980Units":"4553","elevatedPct":"5.59","riskScore":6,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Heard","totalPopulation":"12,670","under20":"3014","medianIncome":"$56,724","pre1980Units":"1720","elevatedPct":"6.33","riskScore":6,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Henry","totalPopulation":"266,895","under20":"70250","medianIncome":"$82,379","pre1980Units":"8856","elevatedPct":"1.59","riskScore":2,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Houston","totalPopulation":"176,372","under20":"47708","medianIncome":"$71,237","pre1980Units":"20193","elevatedPct":"5.14","riskScore":5,"riskLevel":"Medium","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Irwin","totalPopulation":"9,132","under20":"2308","medianIncome":"$49,224","pre1980Units":"1910","elevatedPct":"2.12","riskScore":2,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Jackson","totalPopulation":"97,827","under20":"24120","medianIncome":"$82,138","pre1980Units":"6782","elevatedPct":"4.31","riskScore":4,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Jasper","totalPopulation":"17,407","under20":"4226","medianIncome":"$63,718","pre1980Units":"2080","elevatedPct":"3.42","riskScore":3,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Jeff Davis","totalPopulation":"14,936","under20":"4122","medianIncome":"$46,143","pre1980Units":"2683","elevatedPct":"5.6","riskScore":6,"riskLevel":"Medium","additionalData":"-"},
  {"county":"Jefferson","totalPopulation":"14,961","under20":"3800","medianIncome":"$44,459","pre1980Units":"3762","elevatedPct":"6.47","riskScore":6,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Jenkins","totalPopulation":"8,571","under20":"1905","medianIncome":"$41,830","pre1980Units":"1884","elevatedPct":"9.09","riskScore":9,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Johnson","totalPopulation":"9,424","under20":"1770","medianIncome":"$42,603","pre1980Units":"1827","elevatedPct":"9.88","riskScore":10,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Jones","totalPopulation":"29,883","under20":"7203","medianIncome":"$65,515","pre1980Units":"4110","elevatedPct":"1.78","riskScore":2,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Lamar","totalPopulation":"22,257","under20":"5210","medianIncome":"$61,542","pre1980Units":"3281","elevatedPct":"4.46","riskScore":4,"riskLevel":"Medium","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Lanier","totalPopulation":"11,002","under20":"2702","medianIncome":"$50,671","pre1980Units":"1139","elevatedPct":"5.21","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Laurens","totalPopulation":"50,403","under20":"13518","medianIncome":"$47,918","pre1980Units":"9220","elevatedPct":"5.25","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Lee","totalPopulation":"34,260","under20":"9373","medianIncome":"$84,260","pre1980Units":"2362","elevatedPct":"3.19","riskScore":3,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Liberty","totalPopulation":"71,792","under20":"21137","medianIncome":"$59,807","pre1980Units":"6362","elevatedPct":"1.26","riskScore":1,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Lincoln","totalPopulation":"7,919","under20":"1660","medianIncome":"$69,776","pre1980Units":"1843","elevatedPct":"2.78","riskScore":3,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Long","totalPopulation":"21,992","under20":"5872","medianIncome":"$71,285","pre1980Units":"1038","elevatedPct":"3.31","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Lowndes","totalPopulation":"123,138","under20":"34056","medianIncome":"$52,623","pre1980Units":"16567","elevatedPct":"6.43","riskScore":6,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Lumpkin","totalPopulation":"36,362","under20":"8572","medianIncome":"$71,486","pre1980Units":"2704","elevatedPct":"3.89","riskScore":4,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Macon","totalPopulation":"11,897","under20":"5827","medianIncome":"$40,089","pre1980Units":"4375","elevatedPct":"9.79","riskScore":10,"riskLevel":"High","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Madison","totalPopulation":"33,673","under20":"1961","medianIncome":"$58,789","pre1980Units":"1857","elevatedPct":"8.3","riskScore":8,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Marion","totalPopulation":"7,448","under20":"2312","medianIncome":"$47,174","pre1980Units":"2839","elevatedPct":"5.41","riskScore":5,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"McDuffie","totalPopulation":"22,017","under20":"8177","medianIncome":"$50,593","pre1980Units":"4679","elevatedPct":"7.62","riskScore":8,"riskLevel":"High","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"McIntosh","totalPopulation":"12,137","under20":"1648","medianIncome":"$51,411","pre1980Units":"1102","elevatedPct":"3.76","riskScore":4,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Meriwether","totalPopulation":"21,013","under20":"4746","medianIncome":"$53,019","pre1980Units":"4994","elevatedPct":"10.61","riskScore":10,"riskLevel":"High","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Miller","totalPopulation":"5,693","under20":"1453","medianIncome":"$45,201","pre1980Units":"1676","elevatedPct":"13.04","riskScore":10,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Mitchell","totalPopulation":"20,958","under20":"5296","medianIncome":"$45,296","pre1980Units":"4834","elevatedPct":"2.77","riskScore":3,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Monroe","totalPopulation":"32,937","under20":"7073","medianIncome":"$76,428","pre1980Units":"3184","elevatedPct":"3.23","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Montgomery","totalPopulation":"8,997","under20":"2116","medianIncome":"$48,356","pre1980Units":"1604","elevatedPct":"4.48","riskScore":4,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Morgan","totalPopulation":"22,332","under20":"5182","medianIncome":"$74,340","pre1980Units":"2812","elevatedPct":"4.22","riskScore":4,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Murray","totalPopulation":"42,285","under20":"10565","medianIncome":"$62,109","pre1980Units":"5277","elevatedPct":"3.23","riskScore":3,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Muscogee","totalPopulation":"200,767","under20":"55100","medianIncome":"$53,740","pre1980Units":"49571","elevatedPct":"2.83","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Newton","totalPopulation":"124,901","under20":"33724","medianIncome":"$72,128","pre1980Units":"9011","elevatedPct":"3.71","riskScore":4,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Oconee","totalPopulation":"45,298","under20":"12412","medianIncome":"$114,758","pre1980Units":"3261","elevatedPct":"4.79","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Oglethorpe","totalPopulation":"16,200","under20":"3684","medianIncome":"$62,504","pre1980Units":"2396","elevatedPct":"8.96","riskScore":9,"riskLevel":"High","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Paulding","totalPopulation":"191,986","under20":"50592","medianIncome":"$87,459","pre1980Units":"7122","elevatedPct":"2.85","riskScore":3,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Peach","totalPopulation":"29,443","under20":"7370","medianIncome":"$58,471","pre1980Units":"4409","elevatedPct":"10","riskScore":10,"riskLevel":"High","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Pickens","totalPopulation":"38,257","under20":"7316","medianIncome":"$71,955","pre1980Units":"3473","elevatedPct":"3.52","riskScore":4,"riskLevel":"Low","additionalData":"Increase routine screening of children <6 at pediatric visits and WIC; Public outreach: distribute testing and prevention materials to parents and landlords; Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Pierce","totalPopulation":"20,895","under20":"5497","medianIncome":"$52,695","pre1980Units":"3158","elevatedPct":"3.71","riskScore":4,"riskLevel":"Low","additionalData":"-"},
  {"county":"Pike","totalPopulation":"21,473","under20":"5224","medianIncome":"$80,626","pre1980Units":"2133","elevatedPct":"4.87","riskScore":5,"riskLevel":"Medium","additionalData":"-"},
  {"county":"Polk","totalPopulation":"45,331","under20":"12104","medianIncome":"$54,894","pre1980Units":"9051","elevatedPct":"5.35","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Pulaski","totalPopulation":"10,359","under20":"2005","medianIncome":"$46,975","pre1980Units":"2229","elevatedPct":"7.46","riskScore":7,"riskLevel":"High","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Putnam","totalPopulation":"23,521","under20":"4899","medianIncome":"$62,241","pre1980Units":"2871","elevatedPct":"5.49","riskScore":5,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Quitman","totalPopulation":"2,322","under20":"467","medianIncome":"$41,516","pre1980Units":"529","elevatedPct":"3.85","riskScore":4,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Rabun","totalPopulation":"17,792","under20":"3061","medianIncome":"$63,387","pre1980Units":"2922","elevatedPct":"2.7","riskScore":3,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Randolph","totalPopulation":"5,982","under20":"1418","medianIncome":"$39,188","pre1980Units":"1859","elevatedPct":"18.37","riskScore":18,"riskLevel":"High","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Richmond","totalPopulation":"203,554","under20":"53181","medianIncome":"$50,079","pre1980Units":"47961","elevatedPct":"8.29","riskScore":8,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Rockdale","totalPopulation":"98,071","under20":"25071","medianIncome":"$71,984","pre1980Units":"10216","elevatedPct":"3.14","riskScore":3,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Schley","totalPopulation":"4,584","under20":"1157","medianIncome":"$57,510","pre1980Units":"819","elevatedPct":"1.37","riskScore":1,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Screven","totalPopulation":"14,582","under20":"3226","medianIncome":"$52,713","pre1980Units":"2965","elevatedPct":"3.12","riskScore":3,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Seminole","totalPopulation":"8,988","under20":"1942","medianIncome":"$45,223","pre1980Units":"2242","elevatedPct":"5.26","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Spalding","totalPopulation":"71,822","under20":"17834","medianIncome":"$55,086","pre1980Units":"13242","elevatedPct":"8.25","riskScore":8,"riskLevel":"High","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Stephens","totalPopulation":"28,192","under20":"6624","medianIncome":"$51,980","pre1980Units":"5893","elevatedPct":"4.43","riskScore":4,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Stewart","totalPopulation":"4,814","under20":"682","medianIncome":"$41,158","pre1980Units":"1278","elevatedPct":"4.76","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Sumter","totalPopulation":"28,942","under20":"7657","medianIncome":"$44,677","pre1980Units":"7347","elevatedPct":"2.39","riskScore":2,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Talbot","totalPopulation":"5,640","under20":"1021","medianIncome":"$45,163","pre1980Units":"1414","elevatedPct":"7.46","riskScore":7,"riskLevel":"High","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Taliaferro","totalPopulation":"1,611","under20":"324","medianIncome":"$42,138","pre1980Units":"580","elevatedPct":"14.29","riskScore":10,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Tattnall","totalPopulation":"24,912","under20":"5434","medianIncome":"$47,401","pre1980Units":"4135","elevatedPct":"3.98","riskScore":4,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Taylor","totalPopulation":"7,750","under20":"1671","medianIncome":"$44,035","pre1980Units":"1934","elevatedPct":"1.89","riskScore":2,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Telfair","totalPopulation":"8,004","under20":"2113","medianIncome":"$41,547","pre1980Units":"2593","elevatedPct":"5.41","riskScore":5,"riskLevel":"Medium","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Terrell","totalPopulation":"8,570","under20":"2134","medianIncome":"$53,281","pre1980Units":"2544","elevatedPct":"10.92","riskScore":10,"riskLevel":"High","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Thomas","totalPopulation":"45,845","under20":"11733","medianIncome":"$52,236","pre1980Units":"9672","elevatedPct":"3.04","riskScore":3,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Tift","totalPopulation":"41,992","under20":"11487","medianIncome":"$52,962","pre1980Units":"7762","elevatedPct":"2.87","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Toombs","totalPopulation":"27,494","under20":"7870","medianIncome":"$45,125","pre1980Units":"5915","elevatedPct":"2.99","riskScore":3,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Towns","totalPopulation":"13,183","under20":"2152","medianIncome":"$59,273","pre1980Units":"1637","elevatedPct":"2","riskScore":2,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Treutlen","totalPopulation":"6,305","under20":"1558","medianIncome":"$43,533","pre1980Units":"1453","elevatedPct":"3.51","riskScore":4,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Troup","totalPopulation":"71,732","under20":"18754","medianIncome":"$55,852","pre1980Units":"13565","elevatedPct":"6.77","riskScore":7,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Turner","totalPopulation":"9,055","under20":"2452","medianIncome":"$40,865","pre1980Units":"2036","elevatedPct":"3.3","riskScore":3,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Twiggs","totalPopulation":"7,721","under20":"1581","medianIncome":"$48,772","pre1980Units":"1894","elevatedPct":"8.62","riskScore":9,"riskLevel":"High","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Union","totalPopulation":"28,592","under20":"4489","medianIncome":"$64,588","pre1980Units":"2477","elevatedPct":"0.86","riskScore":1,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Upson","totalPopulation":"28,603","under20":"7010","medianIncome":"$48,061","pre1980Units":"7098","elevatedPct":"4.44","riskScore":4,"riskLevel":"Medium","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Walker","totalPopulation":"70,537","under20":"16027","medianIncome":"$53,407","pre1980Units":"15842","elevatedPct":"1.99","riskScore":2,"riskLevel":"Low","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Walton","totalPopulation":"113,978","under20":"27703","medianIncome":"$83,037","pre1980Units":"8054","elevatedPct":"3.75","riskScore":4,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Ware","totalPopulation":"37,535","under20":"9921","medianIncome":"$48,245","pre1980Units":"8836","elevatedPct":"3.94","riskScore":4,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Warren","totalPopulation":"5,004","under20":"1138","medianIncome":"$44,367","pre1980Units":"1436","elevatedPct":"11.11","riskScore":10,"riskLevel":"High","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Washington","totalPopulation":"19,930","under20":"4632","medianIncome":"$48,462","pre1980Units":"4222","elevatedPct":"25.93","riskScore":10,"riskLevel":"High","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Wayne","totalPopulation":"32,645","under20":"8252","medianIncome":"$51,102","pre1980Units":"4617","elevatedPct":"2.93","riskScore":3,"riskLevel":"Low","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Webster","totalPopulation":"2,337","under20":"442","medianIncome":"$45,987","pre1980Units":"546","elevatedPct":"0","riskScore":0,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Wheeler","totalPopulation":"6,623","under20":"1285","medianIncome":"$35,952","pre1980Units":"1143","elevatedPct":"0","riskScore":0,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"White","totalPopulation":"29,470","under20":"6358","medianIncome":"$62,100","pre1980Units":"2842","elevatedPct":"2.38","riskScore":2,"riskLevel":"Low","additionalData":"Maintain surveillance; promote lead-safe renovations during planned rehab"},
  {"county":"Whitfield","totalPopulation":"104,871","under20":"28248","medianIncome":"$61,680","pre1980Units":"16344","elevatedPct":"3.5","riskScore":4,"riskLevel":"Low","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Wilcox","totalPopulation":"8,835","under20":"1788","medianIncome":"$44,905","pre1980Units":"1639","elevatedPct":"6.25","riskScore":6,"riskLevel":"Medium","additionalData":"Targeted outreach to pediatricians and childcare centers; dust-cleaning campaigns"},
  {"county":"Wilkes","totalPopulation":"9,424","under20":"2143","medianIncome":"$47,377","pre1980Units":"2751","elevatedPct":"5.26","riskScore":5,"riskLevel":"Medium","additionalData":"Deploy mobile testing unit and establish at least one community clinic; Grant-funded lead hazard remediation for rental housing and elderly homes; Mandate landlord inspections and tenant education on lead-safe practices"},
  {"county":"Wilkinson","totalPopulation":"8,821","under20":"2129","medianIncome":"$49,648","pre1980Units":"2016","elevatedPct":"5.26","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"},
  {"county":"Worth","totalPopulation":"19,957","under20":"4839","medianIncome":"$48,956","pre1980Units":"4387","elevatedPct":"4.79","riskScore":5,"riskLevel":"Medium","additionalData":"Targeted home inspections for children with elevated BLL and surrounding households; Free or subsidized lead-safe renovations for high-risk housing units"}
];

// utility: normalize names
const norm = s => (s||'').toString().trim().replace(/ county$/i,'').replace(/\s+/g,' ').toLowerCase();

const statsByName = {};
statsData.forEach(d => statsByName[norm(d.county)] = d);

// map init
const map = L.map('map', { preferCanvas:true }).setView([32.7, -83.45], 6);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',{
  maxZoom: 18, attribution: '&copy; OpenStreetMap contributors'
}).addTo(map);

// color map for risk levels
const colorFor = (lvl) => {
  if(!lvl) return '#999';
  const l = lvl.toString().toLowerCase();
  if(l.includes('high')) return '#ef4444';
  if(l.includes('medium')) return '#f59e0b';
  return '#22c55e';
};

// load US counties geojson and filter to Georgia (state FIPS 13)
const GEOJSON_URL = 'https://raw.githubusercontent.com/plotly/datasets/master/geojson-counties-fips.json';

(async function(){
  let allGeo;
  try {
    const r = await fetch(GEOJSON_URL);
    if(!r.ok) throw new Error('GeoJSON fetch failed: '+r.status);
    allGeo = await r.json();
  } catch(e) {
    alert('Failed to load county boundaries GeoJSON. See console.');
    console.error(e);
    return;
  }

  // filter features to Georgia (id starts with '13')
  const gaFeatures = allGeo.features.filter(f => {
    const id = (f.id||'').toString();
    if(id.startsWith('13')) return true;
    if(f.properties){
      const g = (f.properties.GEOID || '').toString();
      if(g.startsWith('13')) return true;
      const name = (f.properties.NAME || f.properties.name || '').toString().toLowerCase();
      if(name && statsByName[name]) return true;
    }
    return false;
  });

  // markers group
  const markers = L.layerGroup().addTo(map);

  // sidebar list container
  const listDiv = document.getElementById('list');

  const countyEntries = [];

  gaFeatures.forEach((feature) => {
    const props = feature.properties || {};
    const fips = (feature.id || props.GEOID || '').toString().padStart(5,'0');
    const geoName = (props.NAME || props.name || '').toString();
    const key = norm(geoName);
    const stats = statsByName[key] || null;

    // centroid via turf
    let lat=32.7, lon=-83.45;
    try {
      const c = turf.centroid(feature).geometry.coordinates;
      lon = c[0]; lat = c[1];
    } catch(e){
      // fallback bbox center
      if(feature.bbox){
        lon = (feature.bbox[0]+feature.bbox[2])/2;
        lat = (feature.bbox[1]+feature.bbox[3])/2;
      }
    }

    // if stats missing, create placeholder minimal object
    let finalStats = stats;
    if(!finalStats){
      finalStats = {
        county: geoName || ('County ' + fips),
        totalPopulation: 'TBD',
        under20: 'TBD',
        medianIncome: 'TBD',
        pre1980Units: 'TBD',
        elevatedPct: 'TBD',
        riskScore: 'TBD',
        riskLevel: 'Unknown',
        additionalData: 'No data provided'
      };
    }

    const markerColor = colorFor(finalStats.riskLevel);
    const marker = L.circleMarker([lat,lon], {
      radius: 7,
      color: '#222',
      weight: 1,
      fillColor: markerColor,
      fillOpacity: 0.95
    }).addTo(markers);

    // popup HTML
    const popupHtml = `
      <div style="font-family:system-ui;min-width:280px">
        <h3 style="margin:0 0 6px 0">${finalStats.county} County</h3>
        <div style="font-size:13px;color:#222;margin-bottom:8px">
          <strong>Risk score:</strong> ${finalStats.riskScore} / 10 &nbsp; <strong>(${finalStats.riskLevel})</strong><br/>
          <strong>Total population:</strong> ${finalStats.totalPopulation}<br/>
          <strong>Under 20 (2023):</strong> ${finalStats.under20}<br/>
          <strong>Median income (2022):</strong> ${finalStats.medianIncome}<br/>
          <strong>Pre-1980 units:</strong> ${finalStats.pre1980Units}<br/>
          <strong>Elevated %:</strong> ${finalStats.elevatedPct}%<br/>
        </div>
        <div style="font-size:13px"><strong>Recommended actions:</strong>
          <div style="margin-top:6px;color:#333">${finalStats.additionalData}</div>
        </div>
      </div>
    `;
    marker.bindPopup(popupHtml);

    // add list row
    const row = document.createElement('div');
    row.className = 'county-row';
    row.innerHTML = `
      <div style="flex:1;min-width:0">
        <div class="count-name">${finalStats.county} County</div>
        <div style="font-size:12px;color:#666">${finalStats.totalPopulation} • ${finalStats.under20} under 20</div>
      </div>
      <div style="text-align:right">
        <div class="score-badge" style="color:${markerColor}">${finalStats.riskScore}</div>
        <div style="font-size:12px;color:#666">${finalStats.riskLevel}</div>
      </div>
    `;
    row.addEventListener('click', ()=> {
      map.setView([lat,lon], Math.max(8, map.getZoom()+2));
      marker.openPopup();
    });
    listDiv.appendChild(row);

    countyEntries.push({ name: finalStats.county, fips, feature, marker, stats: finalStats });
  });

  // search
  const search = document.getElementById('search');
  search.addEventListener('input', (e)=>{
    const q = (e.target.value||'').trim().toLowerCase();
    listDiv.innerHTML = '';
    countyEntries.forEach(entry=>{
      if(!q || entry.name.toLowerCase().includes(q)){
        const row = document.createElement('div');
        row.className = 'county-row';
        const color = colorFor(entry.stats.riskLevel);
        row.innerHTML = `
          <div style="flex:1;min-width:0">
            <div class="count-name">${entry.name} County</div>
            <div style="font-size:12px;color:#666">${entry.stats.totalPopulation} • ${entry.stats.under20} under 20</div>
          </div>
          <div style="text-align:right">
            <div class="score-badge" style="color:${color}">${entry.stats.riskScore}</div>
            <div style="font-size:12px;color:#666">${entry.stats.riskLevel}</div>
          </div>
        `;
        row.addEventListener('click', ()=> {
          const ll = entry.marker.getLatLng();
          map.setView(ll, Math.max(8, map.getZoom()+2));
          entry.marker.openPopup();
        });
        listDiv.appendChild(row);
      }
    });
  });

  // fit map to all markers if possible
  try {
    const group = L.featureGroup(countyEntries.map(c=>c.marker));
    map.fitBounds(group.getBounds().pad(0.2));
  } catch(e) { /* ignore */ }

  console.log('Loaded', countyEntries.length, 'Georgia counties on the map.');
})();
</script>
</body>
</html>
