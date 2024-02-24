# site-performance-script
To improve the loading time of websiite - code for LCP, FCP, TTFB etc along with the information


# To execute the code watch the video
[]()


# Get the LCP element

**List the Largest Contentful Paint in the console and add a green dotted line around the LCP element.**

```
const po = new PerformanceObserver((list) => {
  let entries = list.getEntries();
 
  entries = dedupe(entries, "startTime");
 
  entries.forEach((item, i) => {
    console.dir(item);
    console.log(
      `${i + 1} current LCP item : ${item.element}: ${item.startTime}`,
    );
    item.element ? (item.element.style = "border: 5px dotted lime;") : "";
  });
 
  const lastEntry = entries[entries.length - 1];
  console.log(`LCP is: ${lastEntry.startTime}`);
});
 
po.observe({ type: "largest-contentful-paint", buffered: true });
 
function dedupe(arr, key) {
  return [...new Map(arr.map((item) => [item[key], item])).values()];
}
```

# Cumulative Layout Shift (CLS)

**This script displays the CLS value when the focus of the browser is switched to another tab, since the CLS is calculated during the lifetime of the page.**

```
let cumulativeLayoutShiftScore = 0;
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      cumulativeLayoutShiftScore += entry.value;
    }
  }
});
 
observer.observe({ type: "layout-shift", buffered: true });
 
document.addEventListener("visibilitychange", () => {
  if (document.visibilityState === "hidden") {
    observer.takeRecords();
    observer.disconnect();
 
    console.log(`CLS: ${cumulativeLayoutShiftScore}`);
  }
});
```

# Largest Contentful Paint Sub-Parts (LCP)

```
const LCP_SUB_PARTS = [
  "Time to first byte",
  "Resource load delay",
  "Resource load time",
  "Element render delay",
];
 
new PerformanceObserver((list) => {
  const lcpEntry = list.getEntries().at(-1);
  const navEntry = performance.getEntriesByType("navigation")[0];
  const lcpResEntry = performance
    .getEntriesByType("resource")
    .filter((e) => e.name === lcpEntry.url)[0];
 
  const ttfb = navEntry.responseStart;
  const lcpRequestStart = Math.max(
    ttfb,
    lcpResEntry ? lcpResEntry.requestStart || lcpResEntry.startTime : 0,
  );
  const lcpResponseEnd = Math.max(
    lcpRequestStart,
    lcpResEntry ? lcpResEntry.responseEnd : 0,
  );
  const lcpRenderTime = Math.max(
    lcpResponseEnd,
    lcpEntry ? lcpEntry.startTime : 0,
  );
 
  LCP_SUB_PARTS.forEach((part) => performance.clearMeasures(part));
 
  const lcpSubPartMeasures = [
    performance.measure(LCP_SUB_PARTS[0], {
      start: 0,
      end: ttfb,
    }),
    performance.measure(LCP_SUB_PARTS[1], {
      start: ttfb,
      end: lcpRequestStart,
    }),
    performance.measure(LCP_SUB_PARTS[2], {
      start: lcpRequestStart,
      end: lcpResponseEnd,
    }),
    performance.measure(LCP_SUB_PARTS[3], {
      start: lcpResponseEnd,
      end: lcpRenderTime,
    }),
  ];
 
  // Log helpful debug information to the console.
  console.log("LCP value: ", lcpRenderTime);
  console.log("LCP element: ", lcpEntry.element, lcpEntry?.url);
  console.table(
    lcpSubPartMeasures.map((measure) => ({
      "LCP sub-part": measure.name,
      "Time (ms)": measure.duration,
      "% of LCP": `${
        Math.round((1000 * measure.duration) / lcpRenderTime) / 10
      }%`,
    })),
  );
}).observe({ type: "largest-contentful-paint", buffered: true });
```



# Web Vitals 

**This code is part of Chrome Extension**

```
const valueToRating = (score) =>
  score <= 200 ? "good" : score <= 500 ? "needs-improvement" : "poor";
 
const COLOR_GOOD = "#0CCE6A";
const COLOR_NEEDS_IMPROVEMENT = "#FFA400";
const COLOR_POOR = "#FF4E42";
const RATING_COLORS = {
  good: COLOR_GOOD,
  "needs-improvement": COLOR_NEEDS_IMPROVEMENT,
  poor: COLOR_POOR,
};
 
const observer = new PerformanceObserver((list) => {
  const interactions = {};
 
  for (const entry of list
    .getEntries()
    .filter((entry) => !entry.interactionId)) {
    interactions[entry.interactionId] = interactions[entry.interactionId] || [];
    interactions[entry.interactionId].push(entry);
  }
 
  // Will report as a single interaction even if parts are in separate frames.
  // Consider splitting by animation frame.
  for (const interaction of Object.values(interactions)) {
    const entry = interaction.reduce((prev, curr) =>
      prev.duration >= curr.duration ? prev : curr,
    );
    const value = entry.duration;
    const rating = valueToRating(value);
 
    const formattedValue = `${value.toFixed(0)} ms`;
    console.groupCollapsed(
      `Interaction tracking snippet %c${formattedValue} (${rating})`,
      `color: ${RATING_COLORS[rating] || "inherit"}`,
    );
    console.log("Interaction target:", entry.target);
 
    for (let entry of interaction) {
      console.log(
        `Interaction event type: %c${entry.name}`,
        "font-family: monospace",
      );
 
      // RenderTime is an estimate, because duration is rounded, and may get rounded down.
      // In rare cases it can be less than processingEnd and that breaks performance.measure().
      // Lets make sure its at least 4ms in those cases so you can just barely see it.
      const adjustedPresentationTime = Math.max(
        entry.processingEnd + 4,
        entry.startTime + entry.duration,
      );
 
      console.table([
        {
          subPartString: "Input delay",
          "Time (ms)": Math.round(entry.processingStart - entry.startTime, 0),
        },
        {
          subPartString: "Processing time",
          "Time (ms)": Math.round(
            entry.processingEnd - entry.processingStart,
            0,
          ),
        },
        {
          subPartString: "Presentation delay",
          "Time (ms)": Math.round(
            adjustedPresentationTime - entry.processingEnd,
            0,
          ),
        },
      ]);
    }
 
    console.groupEnd();
  }
});
 
observer.observe({
  type: "event",
  durationThreshold: 0, // 16 minimum by spec
  buffered: true,
});
```


# Measure the time to first byte, from the server

```
// Measure the time to first byte of all the resources loaded
// https://webperf-snippets.nucliweb.net
new PerformanceObserver((entryList) => {
  const [pageNav] = entryList.getEntriesByType("navigation");
  console.log(`TTFB (ms): ${pageNav.responseStart}`);
}).observe({
  type: "navigation",
  buffered: true,
});
```

# Mesure the TTFB sub-parts

```
(() => {
 
  const formatTime = (time) => {
    //round by 2 decimals, use Math.round() for integer
    return Math.round(time * 100) / 100;
  };
 
  new PerformanceObserver((entryList) => {
    const [pageNav] = entryList.getEntriesByType('navigation');
 
    // timing start times are relative
    const activationStart = pageNav.activationStart || 0;
    const workerStart = Math.max(pageNav.workerStart - activationStart, activationStart);
    const dnsStart = Math.max(pageNav.domainLookupStart - activationStart, workerStart);
    const tcpStart = Math.max(pageNav.connectStart - activationStart, dnsStart);
    const sslStart = Math.max(pageNav.secureConnectionStart - activationStart, tcpStart);
    const requestStart = Math.max(pageNav.requestStart - activationStart, sslStart);
    const responseStart = Math.max(pageNav.responseStart - activationStart, requestStart);
 
    // attribution based on https://www.w3.org/TR/navigation-timing-2/#processing-model
    // use associative array to log the results more readable
    let attributionArray = [];
    attributionArray['Redirect Time'] = {'time in ms':formatTime(workerStart - activationStart)};
    attributionArray['Worker and Cache Time'] = {'time in ms':formatTime(dnsStart - workerStart)};
    attributionArray['DNS Time'] = {'time in ms':formatTime(tcpStart - dnsStart)};
    attributionArray['TCP Time'] = {'time in ms':formatTime(sslStart - tcpStart)};
    attributionArray['SSL Time'] = {'time in ms':formatTime(requestStart - sslStart)};
    attributionArray['Request Time'] = {'time in ms':formatTime(responseStart - requestStart)};
    attributionArray['Total TTFB'] = {'time in ms':formatTime(responseStart - activationStart)};
 
    // log the results
    console.log('%cTime to First Byte ('+formatTime(responseStart - activationStart)+'ms)', 'color: blue; font-weight: bold;');
    console.table(attributionArray);
 
    console.log('%cOrigininal navigation entry', 'color: blue; font-weight: bold;');
    console.log(pageNav);
 
  }).observe({
    type: 'navigation',
    buffered: true
  });
})();
```


# Measure the time to first byte of all the resources loaded

```
// Measure the time to first byte of all the resources loaded
// https://webperf-snippets.nucliweb.net
new PerformanceObserver((entryList) => {
  const entries = entryList.getEntries();
  const resourcesLoaded = [...entries]
    .map((entry) => {
      let obj = {};
      // Some resources may have a responseStart value of 0, due
      // to the resource being cached, or a cross-origin resource
      // being served without a Timing-Allow-Origin header set.
 
      if (entry.responseStart > 0) {
        obj = {
          "TTFB (ms)": +entry.responseStart.toFixed(2),
          Resource: entry.name,
        };
      }
      return Object.keys(obj).length > 0 ? obj : undefined;
    })
    .filter((item) => item !== undefined);
 
  console.table(resourcesLoaded);
}).observe({
  type: "resource",
  buffered: true,
});
```

# Event Processing Time

```
const events = new Map([
  ["connectTime", { start: "connectStart", end: "connectEnd" }],
  ["domainLookupTime", { start: "domainLookupStart", end: "domainLookupEnd" }],
  [
    "DOMContentLoaded",
    { start: "domContentLoadedEventStart", end: "domContentLoadedEventEnd" }
  ],
 
  ["onload", { start: "loadEventStart", end: "loadEventEnd" }]
]);
 
const observer = new PerformanceObserver((list) => {
  const displayTimes = [];
  list.getEntries().forEach((entry) => {
    console.log(entry);
    for (const [key, value] of events) {
      const endValue = entry[value.end];
      const startValue = entry[value.start];
 
      const eventTime = endValue - startValue;
 
      displayTimes.push({
        url: entry.name,
        event: key,
        processingTime: `${eventTime.toFixed(2)} ms`
      });
    }
  });
 
  console.table(displayTimes);
});
 
observer.observe({ type: "navigation", buffered: true });
```

# Find Above The Fold Lazy Loaded Images

```
// List all lazily loaded images above the fold
// https://webperf-snippets.nucliweb.net
function findAboveTheFoldLazyLoadedImages() {
  const lazy = document.querySelectorAll('[loading="lazy"]');
  let lazyImages = [];
  
  lazy.forEach((tag) => {
    const {x, y} = tag.getBoundingClientRect();
    const position = parseInt(tag.getBoundingClientRect().top);
 
    if(x < window.innerWidth && y < window.innerHeight && x !== 0 && y !== 0) {
      lazyImages = [...lazyImages, tag];
      console.log(tag);
    }
  });
 
  if( lazyImages.length === 0 ) {
    console.log(
      `%c Good job, the site does not have any lazily loaded images in the viewport.`,
      "background: #222; color: lightgreen; padding: 0.5ch",
    );
  }
}
 
findAboveTheFoldLazyLoadedImages();
```

# Find render-blocking resources

```
function RenderBlocking({
  startTime,
  duration,
  responseEnd,
  name,
  initiatorType,
}) {
  this.startTime = startTime;
  this.duration = duration;
  this.responseEnd = responseEnd;
  this.name = name;
  this.initiatorType = initiatorType;
}
 
function findRenderBlockingResources() {
  return window.performance
    .getEntriesByType("resource")
    .filter(({ renderBlockingStatus }) => renderBlockingStatus === "blocking")
    .map(
      ({ startTime, duration, responseEnd, name, initiatorType }) =>
        new RenderBlocking({
          startTime,
          duration,
          responseEnd,
          name,
          initiatorType,
        }),
    );
}
 
console.table(findRenderBlockingResources());
```

# First And Third Party Script Info

**List all scripts using PerformanceResourceTiming API and separating them by first and third party**

```
// ex: katespade.com - list firsty party subdomains in HOSTS array
const HOSTS = ["assets.katespade.com"];
 
function getScriptInfo() {
  const resourceListEntries = performance.getEntriesByType("resource");
  // set for first party scripts
  const first = [];
  // set for third party scripts
  const third = [];
 
  resourceListEntries.forEach((resource) => {
    // check for initiator type
    const value = "initiatorType" in resource;
    if (value) {
      if (resource.initiatorType === "script") {
        const { host } = new URL(resource.name);
        // check if resource url host matches location.host = first party script
        if (host === location.host || HOSTS.includes(host)) {
          const json = resource.toJSON();
          first.push({ ...json, type: "First Party" });
        } else {
          // add to third party script
          const json = resource.toJSON();
          third.push({ ...json, type: "Third Party" });
        }
      }
    }
  });
 
  const scripts = {
    firstParty: [{ name: "no data" }],
    thirdParty: [{ name: "no data" }],
  };
 
  if (first.length) {
    scripts.firstParty = first;
  }
 
  if (third.length) {
    scripts.thirdParty = third;
  }
 
  return scripts;
}
 
const { firstParty, thirdParty } = getScriptInfo();
 
console.groupCollapsed("FIRST PARTY SCRIPTS");
console.table(firstParty);
console.groupEnd();
 
console.groupCollapsed("THIRD PARTY SCRIPTS");
console.table(thirdParty);
console.groupEnd();
```

# Inline CSS Info and Size


```
// Wait for the page to fully load
 
function findAllInlineCSS() {
  const convertToKb = (bytes) => bytes / 1000;
  const inlineCSS = document.querySelectorAll("style");
  let totalByteSize = 0;
  for (const css of [...inlineCSS]) {
    const html = css.innerHTML;
    const size = new Blob([html]).size;
    css.byteSizeInKb = convertToKb(size);
    totalByteSize += size;
  }
  // customize table here, can right click on header in console to sort table
  console.table(inlineCSS, [
    "baseURI",
    "parentElement",
    "byteSizeInKb",
    "innerHTML",
  ]);
 
  console.log(`Total size: ${convertToKb(totalByteSize)} kB`);
}
 
findAllInlineCSS();
```

# Inline Script Info & Size

**Find all inline scripts on the page and list the scripts and count. Find the total byte size of all the inline scripts in the console.**

```
function findInlineScripts() {
  const inlineScripts = document.querySelectorAll([
    "script:not([async]):not([defer]):not([src])",
  ]);
  console.log(inlineScripts);
  console.log(`COUNT: ${inlineScripts.length}`);
  let totalByteSize = 0;
  for (const script of [...inlineScripts]) {
    const html = script.innerHTML;
    const size = new Blob([html]).size;
    totalByteSize += size;
  }
  console.log(totalByteSize / 1000 + " kb");
}
 
findInlineScripts();
```

