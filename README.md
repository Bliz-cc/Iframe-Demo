# Widget Integration Guide — Bliz iframe Embed

This guide explains how to embed a Bliz interactive widget (spinning wheel, scratch card, treasure chest, etc.) into any website using an iframe. It covers both desktop and mobile integration patterns, event handling, and customization options.

---

## How It Works

When a user visits your Bliz short link, they are shown an interactive widget — a game, prize reveal, or lead capture experience. You can embed this experience directly inside your own page using an `<iframe>` rather than redirecting the user away.

The widget communicates with the parent page via `window.postMessage`, firing structured events you can listen to and react on — for example, showing a congratulations banner when a user wins a prize.

---

## Getting Your Widget URL

1. Log in to your dashboard at [bliz.cc](https://bliz.cc)
2. Open the link you want to embed
3. Copy the short URL — for example `https://bliz.cc/fRuUl9/w`

You can append query parameters to the URL to configure the widget behavior (see [Query Parameters](#query-parameters) below).

---

## Integration Patterns

There are two standard approaches depending on screen size.

### Desktop — Embedded Frame (Static Load)

On desktop, the widget loads immediately inside a fixed-size container. The recommended size is **850 × 700 px**.

```html
<div style="width: 850px; height: 700px; overflow: hidden; border-radius: 8px;">
  <iframe
    src="https://bliz.cc/YOUR_LINK/w?enableFullscreen=true"
    width="100%"
    height="100%"
    style="border: none;"
    title="Bliz Widget"
    allow="fullscreen"
  ></iframe>
</div>
```

### Mobile — Button + Full-Screen Modal (Lazy Load)

On mobile, embed the widget inside a full-screen modal that opens when the user taps a button. This avoids loading the widget until the user explicitly engages, which improves page load time and mobile UX.

```html
<!-- Trigger button -->
<button id="bliz-open-btn">Tap to Play & Win</button>

<!-- Full-screen modal -->
<div id="bliz-modal" style="display:none; position:fixed; inset:0; background:#fff; z-index:1000; flex-direction:column;">
  <div style="padding:10px 20px; display:flex; justify-content:flex-end; border-bottom:1px solid #ddd;">
    <button id="bliz-close-btn" style="font-size:24px; border:none; background:none; cursor:pointer;">✕</button>
  </div>
  <iframe
    id="bliz-mobile-iframe"
    width="100%"
    style="flex:1; border:none;"
    title="Bliz Widget"
    allow="fullscreen"
  ></iframe>
</div>

<script>
  var WIDGET_URL = "https://bliz.cc/YOUR_LINK/w?enableFullscreen=true";

  document.getElementById('bliz-open-btn').addEventListener('click', function() {
    var iframe = document.getElementById('bliz-mobile-iframe');
    // Lazy load — only set src on first open
    if (!iframe.src) iframe.src = WIDGET_URL;
    document.getElementById('bliz-modal').style.display = 'flex';
  });

  document.getElementById('bliz-close-btn').addEventListener('click', function() {
    document.getElementById('bliz-modal').style.display = 'none';
  });
</script>
```

### Responsive Pattern (Both Combined)

Use CSS media queries to switch between desktop and mobile patterns automatically:

```html
<style>
  .bliz-desktop { display: none; }
  .bliz-mobile  { display: none; }

  @media (min-width: 860px) { .bliz-desktop { display: block; } }
  @media (max-width: 859px) { .bliz-mobile  { display: block; } }
</style>

<div class="bliz-desktop">
  <!-- static 850×700 iframe here -->
</div>

<div class="bliz-mobile">
  <!-- button + modal here -->
</div>
```

---

## Query Parameters

Append these to your widget URL to control behavior:

| Parameter | Type | Description |
|---|---|---|
| `enableFullscreen` | `true` / `false` | Allows the widget to request fullscreen mode. Recommended for mobile. |
| `m` | string | Custom tracking mode or label, passed back in events for your own analytics. |
| `partnerId` | string | Your internal partner or campaign identifier, included in event payloads. |

**Example:**
```
https://bliz.cc/fRuUl9/w?enableFullscreen=true&m=summer_campaign&partnerId=STORE_001
```

---

## Listening to Widget Events

The widget sends messages to the parent page via `window.postMessage`. Listen for them with a standard event listener:

```javascript
window.addEventListener("message", function(event) {
  var data = event.data;

  // Always validate the event shape before processing
  if (!data || !data.widgetId || !data.event) return;

  console.log("Bliz event:", data.event, data);
});
```

### Event Reference

| Event | When it fires | Useful for |
|---|---|---|
| `WIDGET_VIEW` | Widget is loaded and visible | Impression tracking |
| `WIDGET_CONSENT_GAME_START` | User accepts terms and starts the game | Consent logging |
| `WIDGET_INTERACTION` | User is actively engaging (spinning, scratching, etc.) | Progress indicators |
| `WIDGET_INTERACTION_FINISHED` | Game sequence is complete | Triggering post-game UI |
| `WIDGET_SUBMIT` | User submits a lead form inside the widget | CRM / email capture |

### Event Payload Shape

```javascript
{
  widgetId: "abc123",          // Bliz widget identifier
  event: "WIDGET_SUBMIT",      // One of the event names above
  promo: {
    prize: "20% Off Coupon",   // Prize label (on win events)
    code:  "SAVE20"            // Optional promo code
  },
  m: "summer_campaign",        // Your custom `m` parameter if set
  partnerId: "STORE_001"       // Your partnerId if set
}
```

### Practical Example — Show Prize on Win

```javascript
window.addEventListener("message", function(event) {
  var data = event.data;
  if (!data || !data.widgetId || !data.event) return;

  if (data.event === 'WIDGET_SUBMIT' || data.event === 'WIDGET_INTERACTION_FINISHED') {
    var prize = data.promo && data.promo.prize ? data.promo.prize : "a special reward";
    
    // Show your own congratulations UI
    document.getElementById('prize-banner').textContent = "You won: " + prize;
    document.getElementById('prize-banner').style.display = 'block';
  }
});
```

---

## Security Note

Always validate that incoming `postMessage` events originate from Bliz before acting on them:

```javascript
window.addEventListener("message", function(event) {
  // Only trust messages from the Bliz domain
  if (event.origin !== "https://bliz.cc") return;

  var data = event.data;
  if (!data || !data.widgetId || !data.event) return;

  // Safe to process
});
```

---

## Full Working Example

A complete desktop + mobile responsive demo is included in this repository as `widget-demo.html`. Open it in a browser to see both patterns in action. Events are displayed as toast notifications in the top-right corner of the page.

To test locally:

```bash
# Python
python3 -m http.server 8080

# Node
npx serve .
```

Then open `http://localhost:8080/widget-demo.html`.

---

## Checklist

Before going live, confirm the following:

- [ ] Widget URL uses your real Bliz link slug
- [ ] `enableFullscreen=true` is set if embedding on mobile
- [ ] `postMessage` listener validates `event.origin` before acting
- [ ] Mobile modal lazy-loads the iframe (only sets `src` on first open)
- [ ] The container has `overflow: hidden` to prevent iframe scroll bleed
- [ ] You have tested on both desktop and a real mobile device

---

## Support

For questions about your widget URL, prize configuration, or event payloads, visit [bliz.cc](https://bliz.cc) or contact support from within your dashboard.