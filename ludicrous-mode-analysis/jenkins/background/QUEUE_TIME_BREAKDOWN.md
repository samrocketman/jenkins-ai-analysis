# Queue time breakdown (35 seconds total)

## Option A — Two slices (main view)

```mermaid
%%{init: {'theme':'base', 'themeVariables': {
  'pie1':'#b3e5fc',
  'pie2':'#c8e6c9',
  'pie3':'#fff9c4',
  'pie4':'#ffccbc',
  'pie5':'#d1c4e9',
  'pie6':'#f8bbd0',
  'pie7':'#b2dfdb',
  'pie8':'#d7ccc8',
  'pie9':'#cfd8dc',
  'pie10':'#ffecb3',
  'pie11':'#c5e1a5',
  'pie12':'#b2ebf2',
  'pieOuterStrokeWidth':'2',
  'pieStrokeColor':'#e0e0e0',
  'pieOpacity':'1',
  'pieTitleTextSize':'16',
  'pieSectionTextSize':'14',
  'pieLegendTextSize':'14',
  'pieLegendTextColor':'#f5f5f5',
  'pieTitleTextColor':'#f5f5f5',
  'pieSectionColors':'#b3e5fc,#c8e6c9,#fff9c4,#ffccbc,#d1c4e9'
}}}%%
pie showData
    title Total 35 seconds
    "Queue maintenance" : 30
    "Executors take builds from queue" : 5
```

## Option B — All three in one chart (10 ms visible)

Values in milliseconds so the 10 ms slice is visible:

```mermaid
%%{init: {'theme':'base', 'themeVariables': {
  'pie1':'#b3e5fc',
  'pie2':'#c8e6c9',
  'pie3':'#fff9c4',
  'pie4':'#ffccbc',
  'pie5':'#d1c4e9',
  'pieSectionColors':'#b3e5fc,#fff9c4,#c8e6c9',
  'pieStrokeColor':'#e0e0e0',
  'pieOuterStrokeWidth':'2',
  'pieLegendTextColor':'#f5f5f5',
  'pieTitleTextColor':'#f5f5f5'
}}}%%
pie showData
    title Total 35 seconds (values in ms)
    "Queue maintenance (other)" : 29990
    "leastload selects agents for queued work" : 10
    "Executors take builds from queue" : 5000
```

*(30 s = 30 000 ms; 10 ms is the “leastload” slice; 5 s = 5 000 ms.)*
