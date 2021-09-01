# Scenario Azure Virtual Desktop

{{- $pillars := slice "scenario/avd" -}}

{{ partial "scenariolens.partial" (dict "pillarDisplayName" "Azure Virtual Desktop (AVD)" "input" $.Site.Data.input "pillars" $pillars "categories" $.Site.Data.categories) }}