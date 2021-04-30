1. What may be the goal of the following lines of code:
```java
    return text
      .replace(/\u0401/g, 'YO')
      .replace(/\u0419/g, 'I')
      .replace(/\u0426/g, 'TS')
      .replace(/\u0423/g, 'U')
      .replace(/\u041A/g, 'K')
      ...
```
2. Following configuration snippet is in principle something similar that Google Workflows does. Please describe what business logic is executed via this configuration and what could be the intention 
given that XTR is component responsible for making X-Road queries. 

ee.eesti.java.ruuter/configurations/RETS/HK_RETSEPTID.json content:
```json
{
  "action_type": "BLOGIC",
  "destination": [
    {
      "user_info": {
        "verify": {
          "response": {
            "ok": "proceed",
            "nok": "stop"
          }
        },
        "proceed": {
          "method": "get",
          "cookies": [
            "JWTTOKEN"
          ],
          "endpoint": "{tim_userinfo_url}",
          "response": {
            "ok": "proceed",
            "nok": "stop"
          }
        }
      }
    },
    {
      "retseptide_loetelu_patsient": {
        "verify": {
          "response": {
            "ok": "proceed",
            "nok": "stop"
          }
        },
        "proceed": {
          "method": "post",
          "endpoint": "{xtr_get_url}",
          "post_body_struct": {
            "register": "rets",
            "service": "retseptide_loetelu_patsient",
            "parameters": {
              "isikukood": "{#.user_info#.personalCode}",
              "alates": "20.08.1991"
            }
          },
          "response": {
            "ok": "proceed",
            "nok": "stop"
          }
        }
      }
    },
    {
      "rets_retseptide_detailandmed": {
        "verify": {
          "response": {
            "ok": "proceed",
            "nok": "stop"
          }
        },
        "proceed": {
          "method": "post",
          "endpoint": "{xtr_get_url}",
          "multi_request": {
            "field": "retseptinumber",
            "collection": "{#.retseptide_loetelu_patsient#.response#.retsept#.item}"
          },
          "post_body_struct": {
            "register": "rets",
            "service": "retsepti_vaatamine_patsient",
            "parameters": {
              "isikukood": "{#.user_info#.personalCode}",
              "retseptinumber": "{#.retseptinumber}"
            }
          },
          "response": {
            "ok": "proceed",
            "nok": "stop"
          }
        }
      }
    }
  ]
}
```

3.  Describe technological setup and architectural choices made:
 
```java
import 'zone.js/dist/zone-node';

import { ngExpressEngine } from '@nguniversal/express-engine';
import * as express from 'express';
import { join } from 'path';

import { AppServerModule } from './src/main.server';
import { APP_BASE_HREF } from '@angular/common';
import { existsSync } from 'fs';
import { applyLogPrefix, getLogger } from '@stateportal/logger';
import { isDevMode } from '@angular/core';
import NodeCache from 'node-cache';

require('source-map-support').install();

applyLogPrefix();
const log = getLogger('server');
log.setLevel('DEBUG');

export function app(): express.Express {
  const server = express();
  const browserDistFolder = join(process.cwd(), 'dist/apps/stateportal/browser');
  const serverDistFolder = join(process.cwd(), 'dist/apps/stateportal/server');
  const indexHtml = existsSync(join(browserDistFolder, 'index.original.html')) ? 'index.original.html' : 'index';

  // Our Universal express-engine (found @ https://github.com/angular/universal/tree/master/modules/express-engine)
  server.engine(
    'html',
    ngExpressEngine({
      bootstrap: AppServerModule
    })
  );

  server.set('view engine', 'html');
  server.set('views', browserDistFolder);

  server.all('/router/api/**', (req, res) => {
    res.status(404).send('Router api proxy not configured');
  });

  server.get('/healthz', (req, res) => {
    res.sendFile(join(serverDistFolder, 'healthz.json'));
  });

  server.get(['/:lang/dashboard(/*)?', '/:lang/minu(/*)?'], (req, res) =>
    res.sendFile(join(browserDistFolder, 'index.html'))
  );

  server.get(
    '*.*',
    express.static(browserDistFolder, {
      // maxAge: '1y',
      fallthrough: false
    })
  );

  const nodeCache = new NodeCache({ stdTTL: 300, checkperiod: 600, maxKeys: 1000 });
  server.get('*', cacheResponse(nodeCache), (req, res) => {
    res.render(indexHtml, { req, providers: [{ provide: APP_BASE_HREF, useValue: req.baseUrl }] });
  });

  return server;
}

function run(): void {
  const port = process.env.PORT || 4000;
  const server = app();
  server.listen(port, () => {
    log.info(`Node Express server listening on http://localhost:${port}`);
  });
}

function cacheResponse(nodeCache: NodeCache) {
  return (req, res, next) => {
    if (req.method !== 'GET' || isDevMode()) {
      next();
      return;
    }

    const cacheKey = req.originalUrl;
    const cachedResponse = nodeCache.get(cacheKey);
    if (cachedResponse) {
      res.send(cachedResponse);
      return;
    } else {
      res.sendResponse = res.send;
      res.send = body => {
        nodeCache.set(cacheKey, body);
        res.sendResponse(body);
      };
      next();
    }
  };
}
export * from './src/main.server';
```
