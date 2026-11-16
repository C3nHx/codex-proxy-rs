# URL subpath browser evidence

Source head: 459bd614a1f45dcd8cb4d6ac9ee3c2e907ce8ac4
Baseline: 0f5bf03e13a43b34c43b9e3a6dee935ad43f6d2b

All images capture actual built login pages in the same isolated Edge session at 1250 x 625, light theme, with empty credential fields. The local static server provides no mock account or model data.

- before-root.png: upstream baseline build, http://127.0.0.1:8427/login
- after-prefix.png: source head built with --base /proxy/, http://127.0.0.1:8428/proxy/login
- after-root.png: source head built with --base /, http://127.0.0.1:8429/login

These images show layout and asset loading. Login, session renewal and SSE were separately verified in the earlier HTTPS deployment of the identical frontend routing changes. Full model WebSocket and real upstream generation were not tested.
