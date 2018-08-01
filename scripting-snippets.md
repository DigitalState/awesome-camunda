# Scripting Snippets


## Load Config.json into memory and into a Process Variable(optional)

Provides the ability to load into memory a json file stored as a Resource in the process deployment, and then process the json into Process Variables.

### Javascript 

```javascript
/**
 * Load configuration file as a SPIN JSON variable in-memory and optionally as a process variable.
 *
 * @param string fileName The name of the configuration file in the deployment.
 * @param string key The top level JSON key in the configuration file that will be saved, and other keys/objects are omitted.
 * @param boolean persist Whether to save the configuration as a process variable.
 * @return SPIN JSON Object
 */
function loadConfig(fileName, key, persist)
{
  'use strict';

  if (typeof(persist) == 'undefined') {
    persist = false;
  }

  if (typeof(key) == 'undefined') {
    key = null;
  }

  var processDefinitionId = execution.getProcessDefinitionId();
  var deploymentId = execution.getProcessEngineServices().getRepositoryService().getProcessDefinition(processDefinitionId).getDeploymentId();
  var resource = execution.getProcessEngineServices().getRepositoryService().getResourceAsStream(deploymentId, fileName);

  var Scanner = Java.type('java.util.Scanner');

  var scannerResource = new Scanner(resource, 'UTF-8');

  var configText = scannerResource.useDelimiter('\\Z').next();
  scannerResource.close();

  var configAsJson = S(configText);

  if (key === null) {
    var config = configAsJson;
  } else {
    if (!configAsJson.hasProp(key)) {
      throw 'Key "' + key + '" does not exist.';
    }
    var config = configAsJson.prop(key);
  }

  if (persist) {
    execution.setVariable('_config', config);
  }

  return config;
}

loadConfig('config.json', 'myProcess', true);
// loadConfig('config.json');
// loadConfig('config.json', null, true);
// loadConfig('config.json', null, false);
// loadConfig('config.json', 'myprocess');
// loadConfig('config.json', 'myprocess', true);
// loadConfig('config.json', 'myprocess', false);

```


## Render FreeMarker template into memory

Camunda has a 4000 character limit of string variables that are stored in the database.  This creates a issue when trying to process FreeMarker templates: specifically when using a Script Task with a script type of "freemarker" the result variable will be stored as a Process Variable.  For templates that are larger than 4000 characters, this will cause a error in the process.  The workaround for this issue is to execute the freemarker engine in memory/in-scripting.  This snippet provides a easy to use pattern for executing the freemarker template engine in memory and getting the result in memory, allowing you to save the variable in the desired format (such as a SPIN JSON variable or Java Object, which does not have the 4000 character limit).

### Javascript

```javascript
/**
 * Evaluate/Render a FreeMarker template
 *
 * @param string content The string content of a FreeMarker template.
 * @param string object The KeyValue object/JSON object for placeholder bindings.
 * @return string The rendered FreeMarker template.
 */
function renderFreeMarkerTemplate(content, placeholderValues)
{
  // 'use strict' cannot be used at a global script level because JavaImporter's with(){} does not support it.

  var ScriptEngine = new JavaImporter(javax.script);

  with (ScriptEngine) {
    var manager = new ScriptEngineManager();
    var engine = manager.getEngineByName('freemarker');

    var bindings = engine.createBindings();
    bindings.put('placeholders', placeholderValues);

    var rendered = engine.eval(content, bindings);

    return rendered;
  }
}

var placeholderValues = {
   "firstName": "John",
   "lastName": "Smith"
}

var renderedTemplate = renderFreeMarkerTemplate(content, placeholderValues);
// renderedTemplate.toString();
```

where `content` is the string content of a FreeMarker template file.

FreeMarker Template

```
This is a sample FreeMarker template file.
My First Name: ${placeholders.firstName}
My Last Name: ${placeholders.lastName}
```