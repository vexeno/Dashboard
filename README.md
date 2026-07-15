# SeeHex

A minimal Vue/Vite personal navigation page for `seehex.com`.

## Edit links and content

Most homepage content lives in:

```text
public/site.config.json
```

Update that file to change cards, links, email, GitHub, avatar paths, and typing text.

For local preview with the JSON editor enabled, open:

```text
http://localhost:5173/?edit=1
```

The editor is hidden on normal visits.

## Development

```bash
npm install
npm run dev
```

## Build

```bash
npm run build
```

Upload the generated `dist/` folder to a static host.
