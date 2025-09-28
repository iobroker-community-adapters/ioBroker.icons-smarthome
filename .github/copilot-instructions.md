# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

## Adapter-Specific Context

This is the **icons-smarthome** adapter for ioBroker, which provides smart home icons for visualizations.

- **Adapter Name**: iobroker.icons-smarthome
- **Primary Function**: Icon provider for ioBroker visualizations (WebUI, VIS, etc.)
- **Adapter Type**: `visualization-icons` (icon resource adapter)
- **Mode**: `none` (static resource provider, no active instance needed)
- **Key Features**: 
  - Provides smart home icons extracted from ioBroker.habpanel
  - Static icon resource serving via `/www` directory
  - No configuration required (`noConfig: true`)
  - Singleton adapter (`singleton: true`)
  - Only WWW content (`onlyWWW: true`)
- **Original Source**: Icons from Roman Malashkov's Smart House Icon Set (CC0-1.0 license)
- **Repository**: https://github.com/iobroker-community-adapters/ioBroker.icons-smarthome
- **License**: CC0-1.0 (Creative Commons Public Domain)

## Development Context

This adapter is primarily a static resource provider that:
- Serves icon files through the `/www` directory
- Requires no runtime JavaScript execution (`mode: "none"`)
- Provides icons for use in ioBroker visualization adapters
- Uses semantic versioning for icon set updates
- Maintains compatibility with ioBroker visualization systems

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            position: TEST_COORDINATES,
                            createCurrently: true,
                            createHourly: true,
                            createDaily: true,
                            // Add other configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Validate expected states exist
                        const states = await harness.getStatesOf('your-adapter.0');
                        
                        // Check for expected states (adapt based on your adapter)
                        const expectedStates = [
                            'your-adapter.0.info.connection',
                            'your-adapter.0.some.expected.state'
                        ];
                        
                        for (const stateName of expectedStates) {
                            const state = await harness.getState(stateName);
                            if (!state) {
                                return reject(new Error(`Expected state ${stateName} not found`));
                            }
                        }

                        resolve();
                    } catch (error) {
                        reject(error);
                    }
                });
            }).timeout(120000);
        });
    }
});
```

### Practical Testing for Static Resource Adapters

For icon/resource adapters like icons-smarthome, focus testing on:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');
const fs = require('fs');

tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Static Resource Adapter Tests', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should validate www directory structure', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        // Check if www directory exists and contains expected files
                        const wwwPath = path.join(__dirname, '..', 'www');
                        if (!fs.existsSync(wwwPath)) {
                            return reject(new Error('www directory not found'));
                        }

                        // Check for expected icon files or structure
                        const files = fs.readdirSync(wwwPath);
                        if (files.length === 0) {
                            return reject(new Error('www directory is empty'));
                        }

                        console.log(`✅ Found ${files.length} files in www directory`);
                        resolve();
                    } catch (error) {
                        reject(error);
                    }
                });
            });

            it('should have proper io-package.json configuration for resource adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        const ioPackage = require('../io-package.json');
                        
                        // Validate resource adapter configuration
                        if (ioPackage.common.onlyWWW !== true) {
                            return reject(new Error('Resource adapter should have onlyWWW: true'));
                        }
                        
                        if (ioPackage.common.mode !== 'none') {
                            return reject(new Error('Resource adapter should have mode: "none"'));
                        }

                        if (ioPackage.common.type !== 'visualization-icons') {
                            return reject(new Error('Icon adapter should have type: "visualization-icons"'));
                        }

                        console.log('✅ io-package.json configuration validated for resource adapter');
                        resolve();
                    } catch (error) {
                        reject(error);
                    }
                });
            });
        });
    }
});
```

### Package Testing
Current test suite validates:
- Package.json structure and required fields
- io-package.json structure and ioBroker-specific requirements
- Version consistency between package.json and io-package.json
- Proper licensing and repository information

## ioBroker Development Patterns

### Logging
Use ioBroker's built-in logging system:
```javascript
this.log.error('Critical error message');
this.log.warn('Warning message');
this.log.info('Information message');
this.log.debug('Debug message');
```

### State Management
```javascript
// Set state with acknowledgment
await this.setStateAsync('info.connection', true, true);

// Get state
const state = await this.getStateAsync('some.state');
if (state) {
    this.log.info(`State value: ${state.val}`);
}

// Subscribe to state changes
this.subscribeStates('*');
```

### Object Management
```javascript
// Create or update object
await this.setObjectAsync('some.state', {
    type: 'state',
    common: {
        name: 'Some State',
        type: 'boolean',
        role: 'indicator',
        read: true,
        write: false,
    },
    native: {},
});

// Get object
const obj = await this.getObjectAsync('some.state');
```

### Error Handling
```javascript
try {
    // Risky operation
    await this.someAsyncOperation();
} catch (error) {
    this.log.error(`Operation failed: ${error.message}`);
    // Handle error appropriately
}
```

### Adapter Lifecycle
```javascript
class MyAdapter extends utils.Adapter {
    constructor(options = {}) {
        super({
            ...options,
            name: 'my-adapter',
        });
        this.on('ready', this.onReady.bind(this));
        this.on('unload', this.onUnload.bind(this));
    }

    async onReady() {
        // Adapter startup logic
        this.log.info('Adapter started');
    }

    onUnload(callback) {
        try {
            // Clean up resources
            this.log.info('Adapter stopped');
            callback();
        } catch (e) {
            callback();
        }
    }
}
```

### Resource Adapter Patterns

For static resource adapters (like icons-smarthome):

```javascript
// Minimal adapter implementation for resource serving
class IconsAdapter extends utils.Adapter {
    constructor(options = {}) {
        super({
            ...options,
            name: 'icons-smarthome',
        });
        this.on('ready', this.onReady.bind(this));
        this.on('unload', this.onUnload.bind(this));
    }

    async onReady() {
        // For static resource adapters, typically just log and set connection
        this.log.info('Icons adapter ready - serving static resources');
        await this.setStateAsync('info.connection', true, true);
    }

    onUnload(callback) {
        try {
            // Clean shutdown
            this.log.info('Icons adapter stopped');
            callback();
        } catch (e) {
            callback();
        }
    }
}
```

### Configuration Management (JSON Config)

For adapters that need configuration (note: icons-smarthome uses `noConfig: true`):

```javascript
// Access configuration
const config = this.config;
const username = config.username;
const password = config.password;

// Validate configuration
if (!username || !password) {
    this.log.error('Username and password are required');
    return;
}
```

#### JSON Config Schema
```json
{
    "type": "panel",
    "items": {
        "username": {
            "type": "text",
            "label": "Username",
            "placeholder": "Enter username"
        },
        "password": {
            "type": "password",
            "label": "Password",
            "placeholder": "Enter password"
        }
    }
}
```

## Code Style and Standards

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods
- For resource adapters, focus on file structure and accessibility
- Maintain clean `/www` directory organization
- Follow ioBroker naming conventions for states and objects
- Use proper error handling and logging levels

## CI/CD and Testing Integration

### GitHub Actions for Static Resource Testing

For resource adapters like icons-smarthome, implement validation jobs:

```yaml
# Validate static resources and structure
resource-validation:
  runs-on: ubuntu-22.04
  
  steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run package tests
      run: npm run test:package
      
    - name: Validate www directory
      run: |
        if [ ! -d "www" ]; then
          echo "ERROR: www directory not found"
          exit 1
        fi
        
        file_count=$(find www -type f | wc -l)
        if [ "$file_count" -eq 0 ]; then
          echo "ERROR: www directory is empty"
          exit 1
        fi
        
        echo "✅ Found $file_count files in www directory"
```

### CI/CD Best Practices
- Run package validation tests for all adapters
- For resource adapters, validate file structure and accessibility
- Use ubuntu-22.04 for consistency
- Validate licensing and attribution for resource files
- Check file sizes and formats for web compatibility
- Use appropriate timeouts for file validation

### Package.json Script Integration
Standard scripts for ioBroker adapters:
```json
{
  "scripts": {
    "test": "npm run test:package",
    "test:package": "mocha test/package --exit",
    "translate": "translate-adapter",
    "release": "release-script"
  }
}
```

## Documentation Standards

### README.md Structure
```markdown
![Logo](admin/adapter-name.png)
# ioBroker.adapter-name

[![NPM version](badge-url)](npm-url)
[![Downloads](badge-url)](npm-url)
![Test and Release](workflow-badge)

## Description
Brief description of what the adapter does.

## Installation
Installation instructions.

## Configuration  
Configuration details (if applicable).

## Usage
Usage examples and guidelines.

## Changelog
Version history.

## License
License information.
```

### File Structure Standards
```
/
├── .github/
│   ├── workflows/
│   └── copilot-instructions.md
├── admin/
│   ├── adapter-name.png
│   ├── index_m.html (if config needed)
│   └── words.js
├── test/
│   ├── package.js
│   └── mocha.setup.js
├── www/ (for resource adapters)
├── io-package.json
├── main.js (if runtime needed)
├── package.json
├── README.md
└── LICENSE
```

For static resource adapters like icons-smarthome:
- Focus on `/www` directory organization
- No `main.js` needed (mode: "none")
- Minimal configuration (often `noConfig: true`)
- Emphasis on file structure and web accessibility