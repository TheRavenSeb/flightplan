# VTOL VR Flight Planner

## CORS note (why submissions may fail)

This site submits your flight plan to an external endpoint:

```
https://arcane-magica.com/systems/atc/discord
```

Browsers enforce Cross-Origin Resource Sharing (CORS). If the destination server does not include the proper `Access-Control-Allow-Origin` headers or handle the `OPTIONS` preflight request, the browser will block the request.

### What’s implemented here

The app will first try a standard JSON POST. If that fails due to CORS/preflight, it will attempt a CORS-safe fallback using `FormData` without custom headers (which avoids preflight). This often works, but if the backend strictly expects `application/json`, it may still reject the request.

### Recommended fix (server-side)

Enable CORS on the destination server or add a small proxy under a domain you control. Below is a minimal Cloudflare Worker that:

- Responds to `OPTIONS` preflight
- Forwards the JSON `POST` to the upstream
- Returns the upstream response with the correct CORS headers

#### Cloudflare Worker (proxy)

```js
// worker.js
addEventListener('fetch', event => event.respondWith(handle(event.request)));

const UPSTREAM = 'https://arcane-magica.com/systems/atc/discord';
const corsHeaders = {
	'Access-Control-Allow-Origin': 'https://portfolio.theravenseb.site',
	'Access-Control-Allow-Methods': 'POST, OPTIONS',
	'Access-Control-Allow-Headers': 'Content-Type',
};

async function handle(req) {
	if (req.method === 'OPTIONS') {
		return new Response(null, { status: 204, headers: corsHeaders });
	}

	if (req.method === 'POST') {
		const payload = await getPayload(req);
		const upstreamResp = await fetch(UPSTREAM, {
			method: 'POST',
			headers: { 'Content-Type': 'application/json' },
			body: JSON.stringify(payload),
		});

		const bodyText = await upstreamResp.text();
		return new Response(bodyText, { status: upstreamResp.status, headers: corsHeaders });
	}

	return new Response('Method Not Allowed', { status: 405, headers: corsHeaders });
}

async function getPayload(req) {
	const contentType = req.headers.get('content-type') || '';
	if (contentType.includes('application/json')) {
		return await req.json();
	}
	if (contentType.includes('multipart/form-data')) {
		const form = await req.formData();
		const payload = form.get('payload');
		try { return JSON.parse(payload); } catch { return { payload }; }
	}
	const text = await req.text();
	try { return JSON.parse(text); } catch { return { payload: text }; }
}
```

##### Deploy steps (brief)

1. Create a Worker in the Cloudflare dashboard (Workers & Pages → Create).
2. Paste the code above into your Worker.
3. Deploy and bind a route or use the default Worker URL.
4. Update the site’s endpoint in `index.html` to point to your Worker URL.

That’s it—your browser will talk to your Worker (which has proper CORS), and the Worker will forward the request to the original endpoint.

