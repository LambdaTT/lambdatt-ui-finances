# Access Eash Part Individually:

```javascript
import {FINANCES} from 'relative/path/to/lambdatt-ui-finances'

// Components:
FINANCES.COMPONENTS
// Pages:
FINANCES.PAGES
```

# Auto-Wire:

To auto-wire the Finances module, add the following to your `/src/boot/finances-boot.js`:

```javascript
import {FINANCES} from 'relative/path/to/lambdatt-ui-finances'

export default boot(({ app }) => {
    FINANCES.autoWire(app);
})
```