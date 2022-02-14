# index.html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8" />
<style>body{background-color:white;}</style>
<script>(function() {
  // If window.HTMLWidgets is already defined, then use it; otherwise create a
  // new object. This allows preceding code to set options that affect the
  // initialization process (though none currently exist).
  window.HTMLWidgets = window.HTMLWidgets || {};

  // See if we're running in a viewer pane. If not, we're in a web browser.
  var viewerMode = window.HTMLWidgets.viewerMode =
      /\bviewer_pane=1\b/.test(window.location);

  // See if we're running in Shiny mode. If not, it's a static document.
  // Note that static widgets can appear in both Shiny and static modes, but
  // obviously, Shiny widgets can only appear in Shiny apps/documents.
  var shinyMode = window.HTMLWidgets.shinyMode =
      typeof(window.Shiny) !== "undefined" && !!window.Shiny.outputBindings;

  // We can't count on jQuery being available, so we implement our own
  // version if necessary.
  function querySelectorAll(scope, selector) {
    if (typeof(jQuery) !== "undefined" && scope instanceof jQuery) {
      return scope.find(selector);
    }
    if (scope.querySelectorAll) {
      return scope.querySelectorAll(selector);
    }
  }

  function asArray(value) {
    if (value === null)
      return [];
    if ($.isArray(value))
      return value;
    return [value];
  }

  // Implement jQuery's extend
  function extend(target /*, ... */) {
    if (arguments.length == 1) {
      return target;
    }
    for (var i = 1; i < arguments.length; i++) {
      var source = arguments[i];
      for (var prop in source) {
        if (source.hasOwnProperty(prop)) {
          target[prop] = source[prop];
        }
      }
    }
    return target;
  }

  // IE8 doesn't support Array.forEach.
  function forEach(values, callback, thisArg) {
    if (values.forEach) {
      values.forEach(callback, thisArg);
    } else {
      for (var i = 0; i < values.length; i++) {
        callback.call(thisArg, values[i], i, values);
      }
    }
  }

  // Replaces the specified method with the return value of funcSource.
  //
  // Note that funcSource should not BE the new method, it should be a function
  // that RETURNS the new method. funcSource receives a single argument that is
  // the overridden method, it can be called from the new method. The overridden
  // method can be called like a regular function, it has the target permanently
  // bound to it so "this" will work correctly.
  function overrideMethod(target, methodName, funcSource) {
    var superFunc = target[methodName] || function() {};
    var superFuncBound = function() {
      return superFunc.apply(target, arguments);
    };
    target[methodName] = funcSource(superFuncBound);
  }

  // Add a method to delegator that, when invoked, calls
  // delegatee.methodName. If there is no such method on
  // the delegatee, but there was one on delegator before
  // delegateMethod was called, then the original version
  // is invoked instead.
  // For example:
  //
  // var a = {
  //   method1: function() { console.log('a1'); }
  //   method2: function() { console.log('a2'); }
  // };
  // var b = {
  //   method1: function() { console.log('b1'); }
  // };
  // delegateMethod(a, b, "method1");
  // delegateMethod(a, b, "method2");
  // a.method1();
  // a.method2();
  //
  // The output would be "b1", "a2".
  function delegateMethod(delegator, delegatee, methodName) {
    var inherited = delegator[methodName];
    delegator[methodName] = function() {
      var target = delegatee;
      var method = delegatee[methodName];

      // The method doesn't exist on the delegatee. Instead,
      // call the method on the delegator, if it exists.
      if (!method) {
        target = delegator;
        method = inherited;
      }

      if (method) {
        return method.apply(target, arguments);
      }
    };
  }

  // Implement a vague facsimilie of jQuery's data method
  function elementData(el, name, value) {
    if (arguments.length == 2) {
      return el["htmlwidget_data_" + name];
    } else if (arguments.length == 3) {
      el["htmlwidget_data_" + name] = value;
      return el;
    } else {
      throw new Error("Wrong number of arguments for elementData: " +
        arguments.length);
    }
  }

  // http://stackoverflow.com/questions/3446170/escape-string-for-use-in-javascript-regex
  function escapeRegExp(str) {
    return str.replace(/[\-\[\]\/\{\}\(\)\*\+\?\.\\\^\$\|]/g, "\\$&");
  }

  function hasClass(el, className) {
    var re = new RegExp("\\b" + escapeRegExp(className) + "\\b");
    return re.test(el.className);
  }

  // elements - array (or array-like object) of HTML elements
  // className - class name to test for
  // include - if true, only return elements with given className;
  //   if false, only return elements *without* given className
  function filterByClass(elements, className, include) {
    var results = [];
    for (var i = 0; i < elements.length; i++) {
      if (hasClass(elements[i], className) == include)
        results.push(elements[i]);
    }
    return results;
  }

  function on(obj, eventName, func) {
    if (obj.addEventListener) {
      obj.addEventListener(eventName, func, false);
    } else if (obj.attachEvent) {
      obj.attachEvent(eventName, func);
    }
  }

  function off(obj, eventName, func) {
    if (obj.removeEventListener)
      obj.removeEventListener(eventName, func, false);
    else if (obj.detachEvent) {
      obj.detachEvent(eventName, func);
    }
  }

  // Translate array of values to top/right/bottom/left, as usual with
  // the "padding" CSS property
  // https://developer.mozilla.org/en-US/docs/Web/CSS/padding
  function unpackPadding(value) {
    if (typeof(value) === "number")
      value = [value];
    if (value.length === 1) {
      return {top: value[0], right: value[0], bottom: value[0], left: value[0]};
    }
    if (value.length === 2) {
      return {top: value[0], right: value[1], bottom: value[0], left: value[1]};
    }
    if (value.length === 3) {
      return {top: value[0], right: value[1], bottom: value[2], left: value[1]};
    }
    if (value.length === 4) {
      return {top: value[0], right: value[1], bottom: value[2], left: value[3]};
    }
  }

  // Convert an unpacked padding object to a CSS value
  function paddingToCss(paddingObj) {
    return paddingObj.top + "px " + paddingObj.right + "px " + paddingObj.bottom + "px " + paddingObj.left + "px";
  }

  // Makes a number suitable for CSS
  function px(x) {
    if (typeof(x) === "number")
      return x + "px";
    else
      return x;
  }

  // Retrieves runtime widget sizing information for an element.
  // The return value is either null, or an object with fill, padding,
  // defaultWidth, defaultHeight fields.
  function sizingPolicy(el) {
    var sizingEl = document.querySelector("script[data-for='" + el.id + "'][type='application/htmlwidget-sizing']");
    if (!sizingEl)
      return null;
    var sp = JSON.parse(sizingEl.textContent || sizingEl.text || "{}");
    if (viewerMode) {
      return sp.viewer;
    } else {
      return sp.browser;
    }
  }

  // @param tasks Array of strings (or falsy value, in which case no-op).
  //   Each element must be a valid JavaScript expression that yields a
  //   function. Or, can be an array of objects with "code" and "data"
  //   properties; in this case, the "code" property should be a string
  //   of JS that's an expr that yields a function, and "data" should be
  //   an object that will be added as an additional argument when that
  //   function is called.
  // @param target The object that will be "this" for each function
  //   execution.
  // @param args Array of arguments to be passed to the functions. (The
  //   same arguments will be passed to all functions.)
  function evalAndRun(tasks, target, args) {
    if (tasks) {
      forEach(tasks, function(task) {
        var theseArgs = args;
        if (typeof(task) === "object") {
          theseArgs = theseArgs.concat([task.data]);
          task = task.code;
        }
        var taskFunc = tryEval(task);
        if (typeof(taskFunc) !== "function") {
          throw new Error("Task must be a function! Source:\n" + task);
        }
        taskFunc.apply(target, theseArgs);
      });
    }
  }

  // Attempt eval() both with and without enclosing in parentheses.
  // Note that enclosing coerces a function declaration into
  // an expression that eval() can parse
  // (otherwise, a SyntaxError is thrown)
  function tryEval(code) {
    var result = null;
    try {
      result = eval("(" + code + ")");
    } catch(error) {
      if (!error instanceof SyntaxError) {
        throw error;
      }
      try {
        result = eval(code);
      } catch(e) {
        if (e instanceof SyntaxError) {
          throw error;
        } else {
          throw e;
        }
      }
    }
    return result;
  }

  function initSizing(el) {
    var sizing = sizingPolicy(el);
    if (!sizing)
      return;

    var cel = document.getElementById("htmlwidget_container");
    if (!cel)
      return;

    if (typeof(sizing.padding) !== "undefined") {
      document.body.style.margin = "0";
      document.body.style.padding = paddingToCss(unpackPadding(sizing.padding));
    }

    if (sizing.fill) {
      document.body.style.overflow = "hidden";
      document.body.style.width = "100%";
      document.body.style.height = "100%";
      document.documentElement.style.width = "100%";
      document.documentElement.style.height = "100%";
      if (cel) {
        cel.style.position = "absolute";
        var pad = unpackPadding(sizing.padding);
        cel.style.top = pad.top + "px";
        cel.style.right = pad.right + "px";
        cel.style.bottom = pad.bottom + "px";
        cel.style.left = pad.left + "px";
        el.style.width = "100%";
        el.style.height = "100%";
      }

      return {
        getWidth: function() { return cel.offsetWidth; },
        getHeight: function() { return cel.offsetHeight; }
      };

    } else {
      el.style.width = px(sizing.width);
      el.style.height = px(sizing.height);

      return {
        getWidth: function() { return el.offsetWidth; },
        getHeight: function() { return el.offsetHeight; }
      };
    }
  }

  // Default implementations for methods
  var defaults = {
    find: function(scope) {
      return querySelectorAll(scope, "." + this.name);
    },
    renderError: function(el, err) {
      var $el = $(el);

      this.clearError(el);

      // Add all these error classes, as Shiny does
      var errClass = "shiny-output-error";
      if (err.type !== null) {
        // use the classes of the error condition as CSS class names
        errClass = errClass + " " + $.map(asArray(err.type), function(type) {
          return errClass + "-" + type;
        }).join(" ");
      }
      errClass = errClass + " htmlwidgets-error";

      // Is el inline or block? If inline or inline-block, just display:none it
      // and add an inline error.
      var display = $el.css("display");
      $el.data("restore-display-mode", display);

      if (display === "inline" || display === "inline-block") {
        $el.hide();
        if (err.message !== "") {
          var errorSpan = $("<span>").addClass(errClass);
          errorSpan.text(err.message);
          $el.after(errorSpan);
        }
      } else if (display === "block") {
        // If block, add an error just after the el, set visibility:none on the
        // el, and position the error to be on top of the el.
        // Mark it with a unique ID and CSS class so we can remove it later.
        $el.css("visibility", "hidden");
        if (err.message !== "") {
          var errorDiv = $("<div>").addClass(errClass).css("position", "absolute")
            .css("top", el.offsetTop)
            .css("left", el.offsetLeft)
            // setting width can push out the page size, forcing otherwise
            // unnecessary scrollbars to appear and making it impossible for
            // the element to shrink; so use max-width instead
            .css("maxWidth", el.offsetWidth)
            .css("height", el.offsetHeight);
          errorDiv.text(err.message);
          $el.after(errorDiv);

          // Really dumb way to keep the size/position of the error in sync with
          // the parent element as the window is resized or whatever.
          var intId = setInterval(function() {
            if (!errorDiv[0].parentElement) {
              clearInterval(intId);
              return;
            }
            errorDiv
              .css("top", el.offsetTop)
              .css("left", el.offsetLeft)
              .css("maxWidth", el.offsetWidth)
              .css("height", el.offsetHeight);
          }, 500);
        }
      }
    },
    clearError: function(el) {
      var $el = $(el);
      var display = $el.data("restore-display-mode");
      $el.data("restore-display-mode", null);

      if (display === "inline" || display === "inline-block") {
        if (display)
          $el.css("display", display);
        $(el.nextSibling).filter(".htmlwidgets-error").remove();
      } else if (display === "block"){
        $el.css("visibility", "inherit");
        $(el.nextSibling).filter(".htmlwidgets-error").remove();
      }
    },
    sizing: {}
  };

  // Called by widget bindings to register a new type of widget. The definition
  // object can contain the following properties:
  // - name (required) - A string indicating the binding name, which will be
  //   used by default as the CSS classname to look for.
  // - initialize (optional) - A function(el) that will be called once per
  //   widget element; if a value is returned, it will be passed as the third
  //   value to renderValue.
  // - renderValue (required) - A function(el, data, initValue) that will be
  //   called with data. Static contexts will cause this to be called once per
  //   element; Shiny apps will cause this to be called multiple times per
  //   element, as the data changes.
  window.HTMLWidgets.widget = function(definition) {
    if (!definition.name) {
      throw new Error("Widget must have a name");
    }
    if (!definition.type) {
      throw new Error("Widget must have a type");
    }
    // Currently we only support output widgets
    if (definition.type !== "output") {
      throw new Error("Unrecognized widget type '" + definition.type + "'");
    }
    // TODO: Verify that .name is a valid CSS classname

    // Support new-style instance-bound definitions. Old-style class-bound
    // definitions have one widget "object" per widget per type/class of
    // widget; the renderValue and resize methods on such widget objects
    // take el and instance arguments, because the widget object can't
    // store them. New-style instance-bound definitions have one widget
    // object per widget instance; the definition that's passed in doesn't
    // provide renderValue or resize methods at all, just the single method
    //   factory(el, width, height)
    // which returns an object that has renderValue(x) and resize(w, h).
    // This enables a far more natural programming style for the widget
    // author, who can store per-instance state using either OO-style
    // instance fields or functional-style closure variables (I guess this
    // is in contrast to what can only be called C-style pseudo-OO which is
    // what we required before).
    if (definition.factory) {
      definition = createLegacyDefinitionAdapter(definition);
    }

    if (!definition.renderValue) {
      throw new Error("Widget must have a renderValue function");
    }

    // For static rendering (non-Shiny), use a simple widget registration
    // scheme. We also use this scheme for Shiny apps/documents that also
    // contain static widgets.
    window.HTMLWidgets.widgets = window.HTMLWidgets.widgets || [];
    // Merge defaults into the definition; don't mutate the original definition.
    var staticBinding = extend({}, defaults, definition);
    overrideMethod(staticBinding, "find", function(superfunc) {
      return function(scope) {
        var results = superfunc(scope);
        // Filter out Shiny outputs, we only want the static kind
        return filterByClass(results, "html-widget-output", false);
      };
    });
    window.HTMLWidgets.widgets.push(staticBinding);

    if (shinyMode) {
      // Shiny is running. Register the definition with an output binding.
      // The definition itself will not be the output binding, instead
      // we will make an output binding object that delegates to the
      // definition. This is because we foolishly used the same method
      // name (renderValue) for htmlwidgets definition and Shiny bindings
      // but they actually have quite different semantics (the Shiny
      // bindings receive data that includes lots of metadata that it
      // strips off before calling htmlwidgets renderValue). We can't
      // just ignore the difference because in some widgets it's helpful
      // to call this.renderValue() from inside of resize(), and if
      // we're not delegating, then that call will go to the Shiny
      // version instead of the htmlwidgets version.

      // Merge defaults with definition, without mutating either.
      var bindingDef = extend({}, defaults, definition);

      // This object will be our actual Shiny binding.
      var shinyBinding = new Shiny.OutputBinding();

      // With a few exceptions, we'll want to simply use the bindingDef's
      // version of methods if they are available, otherwise fall back to
      // Shiny's defaults. NOTE: If Shiny's output bindings gain additional
      // methods in the future, and we want them to be overrideable by
      // HTMLWidget binding definitions, then we'll need to add them to this
      // list.
      delegateMethod(shinyBinding, bindingDef, "getId");
      delegateMethod(shinyBinding, bindingDef, "onValueChange");
      delegateMethod(shinyBinding, bindingDef, "onValueError");
      delegateMethod(shinyBinding, bindingDef, "renderError");
      delegateMethod(shinyBinding, bindingDef, "clearError");
      delegateMethod(shinyBinding, bindingDef, "showProgress");

      // The find, renderValue, and resize are handled differently, because we
      // want to actually decorate the behavior of the bindingDef methods.

      shinyBinding.find = function(scope) {
        var results = bindingDef.find(scope);

        // Only return elements that are Shiny outputs, not static ones
        var dynamicResults = results.filter(".html-widget-output");

        // It's possible that whatever caused Shiny to think there might be
        // new dynamic outputs, also caused there to be new static outputs.
        // Since there might be lots of different htmlwidgets bindings, we
        // schedule execution for later--no need to staticRender multiple
        // times.
        if (results.length !== dynamicResults.length)
          scheduleStaticRender();

        return dynamicResults;
      };

      // Wrap renderValue to handle initialization, which unfortunately isn't
      // supported natively by Shiny at the time of this writing.

      shinyBinding.renderValue = function(el, data) {
        Shiny.renderDependencies(data.deps);
        // Resolve strings marked as javascript literals to objects
        if (!(data.evals instanceof Array)) data.evals = [data.evals];
        for (var i = 0; data.evals && i < data.evals.length; i++) {
          window.HTMLWidgets.evaluateStringMember(data.x, data.evals[i]);
        }
        if (!bindingDef.renderOnNullValue) {
          if (data.x === null) {
            el.style.visibility = "hidden";
            return;
          } else {
            el.style.visibility = "inherit";
          }
        }
        if (!elementData(el, "initialized")) {
          initSizing(el);

          elementData(el, "initialized", true);
          if (bindingDef.initialize) {
            var result = bindingDef.initialize(el, el.offsetWidth,
              el.offsetHeight);
            elementData(el, "init_result", result);
          }
        }
        bindingDef.renderValue(el, data.x, elementData(el, "init_result"));
        evalAndRun(data.jsHooks.render, elementData(el, "init_result"), [el, data.x]);
      };

      // Only override resize if bindingDef implements it
      if (bindingDef.resize) {
        shinyBinding.resize = function(el, width, height) {
          // Shiny can call resize before initialize/renderValue have been
          // called, which doesn't make sense for widgets.
          if (elementData(el, "initialized")) {
            bindingDef.resize(el, width, height, elementData(el, "init_result"));
          }
        };
      }

      Shiny.outputBindings.register(shinyBinding, bindingDef.name);
    }
  };

  var scheduleStaticRenderTimerId = null;
  function scheduleStaticRender() {
    if (!scheduleStaticRenderTimerId) {
      scheduleStaticRenderTimerId = setTimeout(function() {
        scheduleStaticRenderTimerId = null;
        window.HTMLWidgets.staticRender();
      }, 1);
    }
  }

  // Render static widgets after the document finishes loading
  // Statically render all elements that are of this widget's class
  window.HTMLWidgets.staticRender = function() {
    var bindings = window.HTMLWidgets.widgets || [];
    forEach(bindings, function(binding) {
      var matches = binding.find(document.documentElement);
      forEach(matches, function(el) {
        var sizeObj = initSizing(el, binding);

        if (hasClass(el, "html-widget-static-bound"))
          return;
        el.className = el.className + " html-widget-static-bound";

        var initResult;
        if (binding.initialize) {
          initResult = binding.initialize(el,
            sizeObj ? sizeObj.getWidth() : el.offsetWidth,
            sizeObj ? sizeObj.getHeight() : el.offsetHeight
          );
          elementData(el, "init_result", initResult);
        }

        if (binding.resize) {
          var lastSize = {
            w: sizeObj ? sizeObj.getWidth() : el.offsetWidth,
            h: sizeObj ? sizeObj.getHeight() : el.offsetHeight
          };
          var resizeHandler = function(e) {
            var size = {
              w: sizeObj ? sizeObj.getWidth() : el.offsetWidth,
              h: sizeObj ? sizeObj.getHeight() : el.offsetHeight
            };
            if (size.w === 0 && size.h === 0)
              return;
            if (size.w === lastSize.w && size.h === lastSize.h)
              return;
            lastSize = size;
            binding.resize(el, size.w, size.h, initResult);
          };

          on(window, "resize", resizeHandler);

          // This is needed for cases where we're running in a Shiny
          // app, but the widget itself is not a Shiny output, but
          // rather a simple static widget. One example of this is
          // an rmarkdown document that has runtime:shiny and widget
          // that isn't in a render function. Shiny only knows to
          // call resize handlers for Shiny outputs, not for static
          // widgets, so we do it ourselves.
          if (window.jQuery) {
            window.jQuery(document).on(
              "shown.htmlwidgets shown.bs.tab.htmlwidgets shown.bs.collapse.htmlwidgets",
              resizeHandler
            );
            window.jQuery(document).on(
              "hidden.htmlwidgets hidden.bs.tab.htmlwidgets hidden.bs.collapse.htmlwidgets",
              resizeHandler
            );
          }

          // This is needed for the specific case of ioslides, which
          // flips slides between display:none and display:block.
          // Ideally we would not have to have ioslide-specific code
          // here, but rather have ioslides raise a generic event,
          // but the rmarkdown package just went to CRAN so the
          // window to getting that fixed may be long.
          if (window.addEventListener) {
            // It's OK to limit this to window.addEventListener
            // browsers because ioslides itself only supports
            // such browsers.
            on(document, "slideenter", resizeHandler);
            on(document, "slideleave", resizeHandler);
          }
        }

        var scriptData = document.querySelector("script[data-for='" + el.id + "'][type='application/json']");
        if (scriptData) {
          var data = JSON.parse(scriptData.textContent || scriptData.text);
          // Resolve strings marked as javascript literals to objects
          if (!(data.evals instanceof Array)) data.evals = [data.evals];
          for (var k = 0; data.evals && k < data.evals.length; k++) {
            window.HTMLWidgets.evaluateStringMember(data.x, data.evals[k]);
          }
          binding.renderValue(el, data.x, initResult);
          evalAndRun(data.jsHooks.render, initResult, [el, data.x]);
        }
      });
    });

    invokePostRenderHandlers();
  }


  function has_jQuery3() {
    if (!window.jQuery) {
      return false;
    }
    var $version = window.jQuery.fn.jquery;
    var $major_version = parseInt($version.split(".")[0]);
    return $major_version >= 3;
  }

  /*
  / Shiny 1.4 bumped jQuery from 1.x to 3.x which means jQuery's
  / on-ready handler (i.e., $(fn)) is now asyncronous (i.e., it now
  / really means $(setTimeout(fn)).
  / https://jquery.com/upgrade-guide/3.0/#breaking-change-document-ready-handlers-are-now-asynchronous
  /
  / Since Shiny uses $() to schedule initShiny, shiny>=1.4 calls initShiny
  / one tick later than it did before, which means staticRender() is
  / called renderValue() earlier than (advanced) widget authors might be expecting.
  / https://github.com/rstudio/shiny/issues/2630
  /
  / For a concrete example, leaflet has some methods (e.g., updateBounds)
  / which reference Shiny methods registered in initShiny (e.g., setInputValue).
  / Since leaflet is privy to this life-cycle, it knows to use setTimeout() to
  / delay execution of those methods (until Shiny methods are ready)
  / https://github.com/rstudio/leaflet/blob/18ec981/javascript/src/index.js#L266-L268
  /
  / Ideally widget authors wouldn't need to use this setTimeout() hack that
  / leaflet uses to call Shiny methods on a staticRender(). In the long run,
  / the logic initShiny should be broken up so that method registration happens
  / right away, but binding happens later.
  */
  function maybeStaticRenderLater() {
    if (shinyMode && has_jQuery3()) {
      window.jQuery(window.HTMLWidgets.staticRender);
    } else {
      window.HTMLWidgets.staticRender();
    }
  }

  if (document.addEventListener) {
    document.addEventListener("DOMContentLoaded", function() {
      document.removeEventListener("DOMContentLoaded", arguments.callee, false);
      maybeStaticRenderLater();
    }, false);
  } else if (document.attachEvent) {
    document.attachEvent("onreadystatechange", function() {
      if (document.readyState === "complete") {
        document.detachEvent("onreadystatechange", arguments.callee);
        maybeStaticRenderLater();
      }
    });
  }


  window.HTMLWidgets.getAttachmentUrl = function(depname, key) {
    // If no key, default to the first item
    if (typeof(key) === "undefined")
      key = 1;

    var link = document.getElementById(depname + "-" + key + "-attachment");
    if (!link) {
      throw new Error("Attachment " + depname + "/" + key + " not found in document");
    }
    return link.getAttribute("href");
  };

  window.HTMLWidgets.dataframeToD3 = function(df) {
    var names = [];
    var length;
    for (var name in df) {
        if (df.hasOwnProperty(name))
            names.push(name);
        if (typeof(df[name]) !== "object" || typeof(df[name].length) === "undefined") {
            throw new Error("All fields must be arrays");
        } else if (typeof(length) !== "undefined" && length !== df[name].length) {
            throw new Error("All fields must be arrays of the same length");
        }
        length = df[name].length;
    }
    var results = [];
    var item;
    for (var row = 0; row < length; row++) {
        item = {};
        for (var col = 0; col < names.length; col++) {
            item[names[col]] = df[names[col]][row];
        }
        results.push(item);
    }
    return results;
  };

  window.HTMLWidgets.transposeArray2D = function(array) {
      if (array.length === 0) return array;
      var newArray = array[0].map(function(col, i) {
          return array.map(function(row) {
              return row[i]
          })
      });
      return newArray;
  };
  // Split value at splitChar, but allow splitChar to be escaped
  // using escapeChar. Any other characters escaped by escapeChar
  // will be included as usual (including escapeChar itself).
  function splitWithEscape(value, splitChar, escapeChar) {
    var results = [];
    var escapeMode = false;
    var currentResult = "";
    for (var pos = 0; pos < value.length; pos++) {
      if (!escapeMode) {
        if (value[pos] === splitChar) {
          results.push(currentResult);
          currentResult = "";
        } else if (value[pos] === escapeChar) {
          escapeMode = true;
        } else {
          currentResult += value[pos];
        }
      } else {
        currentResult += value[pos];
        escapeMode = false;
      }
    }
    if (currentResult !== "") {
      results.push(currentResult);
    }
    return results;
  }
  // Function authored by Yihui/JJ Allaire
  window.HTMLWidgets.evaluateStringMember = function(o, member) {
    var parts = splitWithEscape(member, '.', '\\');
    for (var i = 0, l = parts.length; i < l; i++) {
      var part = parts[i];
      // part may be a character or 'numeric' member name
      if (o !== null && typeof o === "object" && part in o) {
        if (i == (l - 1)) { // if we are at the end of the line then evalulate
          if (typeof o[part] === "string")
            o[part] = tryEval(o[part]);
        } else { // otherwise continue to next embedded object
          o = o[part];
        }
      }
    }
  };

  // Retrieve the HTMLWidget instance (i.e. the return value of an
  // HTMLWidget binding's initialize() or factory() function)
  // associated with an element, or null if none.
  window.HTMLWidgets.getInstance = function(el) {
    return elementData(el, "init_result");
  };

  // Finds the first element in the scope that matches the selector,
  // and returns the HTMLWidget instance (i.e. the return value of
  // an HTMLWidget binding's initialize() or factory() function)
  // associated with that element, if any. If no element matches the
  // selector, or the first matching element has no HTMLWidget
  // instance associated with it, then null is returned.
  //
  // The scope argument is optional, and defaults to window.document.
  window.HTMLWidgets.find = function(scope, selector) {
    if (arguments.length == 1) {
      selector = scope;
      scope = document;
    }

    var el = scope.querySelector(selector);
    if (el === null) {
      return null;
    } else {
      return window.HTMLWidgets.getInstance(el);
    }
  };

  // Finds all elements in the scope that match the selector, and
  // returns the HTMLWidget instances (i.e. the return values of
  // an HTMLWidget binding's initialize() or factory() function)
  // associated with the elements, in an array. If elements that
  // match the selector don't have an associated HTMLWidget
  // instance, the returned array will contain nulls.
  //
  // The scope argument is optional, and defaults to window.document.
  window.HTMLWidgets.findAll = function(scope, selector) {
    if (arguments.length == 1) {
      selector = scope;
      scope = document;
    }

    var nodes = scope.querySelectorAll(selector);
    var results = [];
    for (var i = 0; i < nodes.length; i++) {
      results.push(window.HTMLWidgets.getInstance(nodes[i]));
    }
    return results;
  };

  var postRenderHandlers = [];
  function invokePostRenderHandlers() {
    while (postRenderHandlers.length) {
      var handler = postRenderHandlers.shift();
      if (handler) {
        handler();
      }
    }
  }

  // Register the given callback function to be invoked after the
  // next time static widgets are rendered.
  window.HTMLWidgets.addPostRenderHandler = function(callback) {
    postRenderHandlers.push(callback);
  };

  // Takes a new-style instance-bound definition, and returns an
  // old-style class-bound definition. This saves us from having
  // to rewrite all the logic in this file to accomodate both
  // types of definitions.
  function createLegacyDefinitionAdapter(defn) {
    var result = {
      name: defn.name,
      type: defn.type,
      initialize: function(el, width, height) {
        return defn.factory(el, width, height);
      },
      renderValue: function(el, x, instance) {
        return instance.renderValue(x);
      },
      resize: function(el, width, height, instance) {
        return instance.resize(width, height);
      }
    };

    if (defn.find)
      result.find = defn.find;
    if (defn.renderError)
      result.renderError = defn.renderError;
    if (defn.clearError)
      result.clearError = defn.clearError;

    return result;
  }
})();

</script>
<script>/*! jQuery v1.12.4 | (c) jQuery Foundation | jquery.org/license */
!function(a,b){"object"==typeof module&&"object"==typeof module.exports?module.exports=a.document?b(a,!0):function(a){if(!a.document)throw new Error("jQuery requires a window with a document");return b(a)}:b(a)}("undefined"!=typeof window?window:this,function(a,b){var c=[],d=a.document,e=c.slice,f=c.concat,g=c.push,h=c.indexOf,i={},j=i.toString,k=i.hasOwnProperty,l={},m="1.12.4",n=function(a,b){return new n.fn.init(a,b)},o=/^[\s\uFEFF\xA0]+|[\s\uFEFF\xA0]+$/g,p=/^-ms-/,q=/-([\da-z])/gi,r=function(a,b){return b.toUpperCase()};n.fn=n.prototype={jquery:m,constructor:n,selector:"",length:0,toArray:function(){return e.call(this)},get:function(a){return null!=a?0>a?this[a+this.length]:this[a]:e.call(this)},pushStack:function(a){var b=n.merge(this.constructor(),a);return b.prevObject=this,b.context=this.context,b},each:function(a){return n.each(this,a)},map:function(a){return this.pushStack(n.map(this,function(b,c){return a.call(b,c,b)}))},slice:function(){return this.pushStack(e.apply(this,arguments))},first:function(){return this.eq(0)},last:function(){return this.eq(-1)},eq:function(a){var b=this.length,c=+a+(0>a?b:0);return this.pushStack(c>=0&&b>c?[this[c]]:[])},end:function(){return this.prevObject||this.constructor()},push:g,sort:c.sort,splice:c.splice},n.extend=n.fn.extend=function(){var a,b,c,d,e,f,g=arguments[0]||{},h=1,i=arguments.length,j=!1;for("boolean"==typeof g&&(j=g,g=arguments[h]||{},h++),"object"==typeof g||n.isFunction(g)||(g={}),h===i&&(g=this,h--);i>h;h++)if(null!=(e=arguments[h]))for(d in e)a=g[d],c=e[d],g!==c&&(j&&c&&(n.isPlainObject(c)||(b=n.isArray(c)))?(b?(b=!1,f=a&&n.isArray(a)?a:[]):f=a&&n.isPlainObject(a)?a:{},g[d]=n.extend(j,f,c)):void 0!==c&&(g[d]=c));return g},n.extend({expando:"jQuery"+(m+Math.random()).replace(/\D/g,""),isReady:!0,error:function(a){throw new Error(a)},noop:function(){},isFunction:function(a){return"function"===n.type(a)},isArray:Array.isArray||function(a){return"array"===n.type(a)},isWindow:function(a){return null!=a&&a==a.window},isNumeric:function(a){var b=a&&a.toString();return!n.isArray(a)&&b-parseFloat(b)+1>=0},isEmptyObject:function(a){var b;for(b in a)return!1;return!0},isPlainObject:function(a){var b;if(!a||"object"!==n.type(a)||a.nodeType||n.isWindow(a))return!1;try{if(a.constructor&&!k.call(a,"constructor")&&!k.call(a.constructor.prototype,"isPrototypeOf"))return!1}catch(c){return!1}if(!l.ownFirst)for(b in a)return k.call(a,b);for(b in a);return void 0===b||k.call(a,b)},type:function(a){return null==a?a+"":"object"==typeof a||"function"==typeof a?i[j.call(a)]||"object":typeof a},globalEval:function(b){b&&n.trim(b)&&(a.execScript||function(b){a.eval.call(a,b)})(b)},camelCase:function(a){return a.replace(p,"ms-").replace(q,r)},nodeName:function(a,b){return a.nodeName&&a.nodeName.toLowerCase()===b.toLowerCase()},each:function(a,b){var c,d=0;if(s(a)){for(c=a.length;c>d;d++)if(b.call(a[d],d,a[d])===!1)break}else for(d in a)if(b.call(a[d],d,a[d])===!1)break;return a},trim:function(a){return null==a?"":(a+"").replace(o,"")},makeArray:function(a,b){var c=b||[];return null!=a&&(s(Object(a))?n.merge(c,"string"==typeof a?[a]:a):g.call(c,a)),c},inArray:function(a,b,c){var d;if(b){if(h)return h.call(b,a,c);for(d=b.length,c=c?0>c?Math.max(0,d+c):c:0;d>c;c++)if(c in b&&b[c]===a)return c}return-1},merge:function(a,b){var c=+b.length,d=0,e=a.length;while(c>d)a[e++]=b[d++];if(c!==c)while(void 0!==b[d])a[e++]=b[d++];return a.length=e,a},grep:function(a,b,c){for(var d,e=[],f=0,g=a.length,h=!c;g>f;f++)d=!b(a[f],f),d!==h&&e.push(a[f]);return e},map:function(a,b,c){var d,e,g=0,h=[];if(s(a))for(d=a.length;d>g;g++)e=b(a[g],g,c),null!=e&&h.push(e);else for(g in a)e=b(a[g],g,c),null!=e&&h.push(e);return f.apply([],h)},guid:1,proxy:function(a,b){var c,d,f;return"string"==typeof b&&(f=a[b],b=a,a=f),n.isFunction(a)?(c=e.call(arguments,2),d=function(){return a.apply(b||this,c.concat(e.call(arguments)))},d.guid=a.guid=a.guid||n.guid++,d):void 0},now:function(){return+new Date},support:l}),"function"==typeof Symbol&&(n.fn[Symbol.iterator]=c[Symbol.iterator]),n.each("Boolean Number String Function Array Date RegExp Object Error Symbol".split(" "),function(a,b){i["[object "+b+"]"]=b.toLowerCase()});function s(a){var b=!!a&&"length"in a&&a.length,c=n.type(a);return"function"===c||n.isWindow(a)?!1:"array"===c||0===b||"number"==typeof b&&b>0&&b-1 in a}var t=function(a){var b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u="sizzle"+1*new Date,v=a.document,w=0,x=0,y=ga(),z=ga(),A=ga(),B=function(a,b){return a===b&&(l=!0),0},C=1<<31,D={}.hasOwnProperty,E=[],F=E.pop,G=E.push,H=E.push,I=E.slice,J=function(a,b){for(var c=0,d=a.length;d>c;c++)if(a[c]===b)return c;return-1},K="checked|selected|async|autofocus|autoplay|controls|defer|disabled|hidden|ismap|loop|multiple|open|readonly|required|scoped",L="[\\x20\\t\\r\\n\\f]",M="(?:\\\\.|[\\w-]|[^\\x00-\\xa0])+",N="\\["+L+"*("+M+")(?:"+L+"*([*^$|!~]?=)"+L+"*(?:'((?:\\\\.|[^\\\\'])*)'|\"((?:\\\\.|[^\\\\\"])*)\"|("+M+"))|)"+L+"*\\]",O=":("+M+")(?:\\((('((?:\\\\.|[^\\\\'])*)'|\"((?:\\\\.|[^\\\\\"])*)\")|((?:\\\\.|[^\\\\()[\\]]|"+N+")*)|.*)\\)|)",P=new RegExp(L+"+","g"),Q=new RegExp("^"+L+"+|((?:^|[^\\\\])(?:\\\\.)*)"+L+"+$","g"),R=new RegExp("^"+L+"*,"+L+"*"),S=new RegExp("^"+L+"*([>+~]|"+L+")"+L+"*"),T=new RegExp("="+L+"*([^\\]'\"]*?)"+L+"*\\]","g"),U=new RegExp(O),V=new RegExp("^"+M+"$"),W={ID:new RegExp("^#("+M+")"),CLASS:new RegExp("^\\.("+M+")"),TAG:new RegExp("^("+M+"|[*])"),ATTR:new RegExp("^"+N),PSEUDO:new RegExp("^"+O),CHILD:new RegExp("^:(only|first|last|nth|nth-last)-(child|of-type)(?:\\("+L+"*(even|odd|(([+-]|)(\\d*)n|)"+L+"*(?:([+-]|)"+L+"*(\\d+)|))"+L+"*\\)|)","i"),bool:new RegExp("^(?:"+K+")$","i"),needsContext:new RegExp("^"+L+"*[>+~]|:(even|odd|eq|gt|lt|nth|first|last)(?:\\("+L+"*((?:-\\d)?\\d*)"+L+"*\\)|)(?=[^-]|$)","i")},X=/^(?:input|select|textarea|button)$/i,Y=/^h\d$/i,Z=/^[^{]+\{\s*\[native \w/,$=/^(?:#([\w-]+)|(\w+)|\.([\w-]+))$/,_=/[+~]/,aa=/'|\\/g,ba=new RegExp("\\\\([\\da-f]{1,6}"+L+"?|("+L+")|.)","ig"),ca=function(a,b,c){var d="0x"+b-65536;return d!==d||c?b:0>d?String.fromCharCode(d+65536):String.fromCharCode(d>>10|55296,1023&d|56320)},da=function(){m()};try{H.apply(E=I.call(v.childNodes),v.childNodes),E[v.childNodes.length].nodeType}catch(ea){H={apply:E.length?function(a,b){G.apply(a,I.call(b))}:function(a,b){var c=a.length,d=0;while(a[c++]=b[d++]);a.length=c-1}}}function fa(a,b,d,e){var f,h,j,k,l,o,r,s,w=b&&b.ownerDocument,x=b?b.nodeType:9;if(d=d||[],"string"!=typeof a||!a||1!==x&&9!==x&&11!==x)return d;if(!e&&((b?b.ownerDocument||b:v)!==n&&m(b),b=b||n,p)){if(11!==x&&(o=$.exec(a)))if(f=o[1]){if(9===x){if(!(j=b.getElementById(f)))return d;if(j.id===f)return d.push(j),d}else if(w&&(j=w.getElementById(f))&&t(b,j)&&j.id===f)return d.push(j),d}else{if(o[2])return H.apply(d,b.getElementsByTagName(a)),d;if((f=o[3])&&c.getElementsByClassName&&b.getElementsByClassName)return H.apply(d,b.getElementsByClassName(f)),d}if(c.qsa&&!A[a+" "]&&(!q||!q.test(a))){if(1!==x)w=b,s=a;else if("object"!==b.nodeName.toLowerCase()){(k=b.getAttribute("id"))?k=k.replace(aa,"\\$&"):b.setAttribute("id",k=u),r=g(a),h=r.length,l=V.test(k)?"#"+k:"[id='"+k+"']";while(h--)r[h]=l+" "+qa(r[h]);s=r.join(","),w=_.test(a)&&oa(b.parentNode)||b}if(s)try{return H.apply(d,w.querySelectorAll(s)),d}catch(y){}finally{k===u&&b.removeAttribute("id")}}}return i(a.replace(Q,"$1"),b,d,e)}function ga(){var a=[];function b(c,e){return a.push(c+" ")>d.cacheLength&&delete b[a.shift()],b[c+" "]=e}return b}function ha(a){return a[u]=!0,a}function ia(a){var b=n.createElement("div");try{return!!a(b)}catch(c){return!1}finally{b.parentNode&&b.parentNode.removeChild(b),b=null}}function ja(a,b){var c=a.split("|"),e=c.length;while(e--)d.attrHandle[c[e]]=b}function ka(a,b){var c=b&&a,d=c&&1===a.nodeType&&1===b.nodeType&&(~b.sourceIndex||C)-(~a.sourceIndex||C);if(d)return d;if(c)while(c=c.nextSibling)if(c===b)return-1;return a?1:-1}function la(a){return function(b){var c=b.nodeName.toLowerCase();return"input"===c&&b.type===a}}function ma(a){return function(b){var c=b.nodeName.toLowerCase();return("input"===c||"button"===c)&&b.type===a}}function na(a){return ha(function(b){return b=+b,ha(function(c,d){var e,f=a([],c.length,b),g=f.length;while(g--)c[e=f[g]]&&(c[e]=!(d[e]=c[e]))})})}function oa(a){return a&&"undefined"!=typeof a.getElementsByTagName&&a}c=fa.support={},f=fa.isXML=function(a){var b=a&&(a.ownerDocument||a).documentElement;return b?"HTML"!==b.nodeName:!1},m=fa.setDocument=function(a){var b,e,g=a?a.ownerDocument||a:v;return g!==n&&9===g.nodeType&&g.documentElement?(n=g,o=n.documentElement,p=!f(n),(e=n.defaultView)&&e.top!==e&&(e.addEventListener?e.addEventListener("unload",da,!1):e.attachEvent&&e.attachEvent("onunload",da)),c.attributes=ia(function(a){return a.className="i",!a.getAttribute("className")}),c.getElementsByTagName=ia(function(a){return a.appendChild(n.createComment("")),!a.getElementsByTagName("*").length}),c.getElementsByClassName=Z.test(n.getElementsByClassName),c.getById=ia(function(a){return o.appendChild(a).id=u,!n.getElementsByName||!n.getElementsByName(u).length}),c.getById?(d.find.ID=function(a,b){if("undefined"!=typeof b.getElementById&&p){var c=b.getElementById(a);return c?[c]:[]}},d.filter.ID=function(a){var b=a.replace(ba,ca);return function(a){return a.getAttribute("id")===b}}):(delete d.find.ID,d.filter.ID=function(a){var b=a.replace(ba,ca);return function(a){var c="undefined"!=typeof a.getAttributeNode&&a.getAttributeNode("id");return c&&c.value===b}}),d.find.TAG=c.getElementsByTagName?function(a,b){return"undefined"!=typeof b.getElementsByTagName?b.getElementsByTagName(a):c.qsa?b.querySelectorAll(a):void 0}:function(a,b){var c,d=[],e=0,f=b.getElementsByTagName(a);if("*"===a){while(c=f[e++])1===c.nodeType&&d.push(c);return d}return f},d.find.CLASS=c.getElementsByClassName&&function(a,b){return"undefined"!=typeof b.getElementsByClassName&&p?b.getElementsByClassName(a):void 0},r=[],q=[],(c.qsa=Z.test(n.querySelectorAll))&&(ia(function(a){o.appendChild(a).innerHTML="<a id='"+u+"'></a><select id='"+u+"-\r\\' msallowcapture=''><option selected=''></option></select>",a.querySelectorAll("[msallowcapture^='']").length&&q.push("[*^$]="+L+"*(?:''|\"\")"),a.querySelectorAll("[selected]").length||q.push("\\["+L+"*(?:value|"+K+")"),a.querySelectorAll("[id~="+u+"-]").length||q.push("~="),a.querySelectorAll(":checked").length||q.push(":checked"),a.querySelectorAll("a#"+u+"+*").length||q.push(".#.+[+~]")}),ia(function(a){var b=n.createElement("input");b.setAttribute("type","hidden"),a.appendChild(b).setAttribute("name","D"),a.querySelectorAll("[name=d]").length&&q.push("name"+L+"*[*^$|!~]?="),a.querySelectorAll(":enabled").length||q.push(":enabled",":disabled"),a.querySelectorAll("*,:x"),q.push(",.*:")})),(c.matchesSelector=Z.test(s=o.matches||o.webkitMatchesSelector||o.mozMatchesSelector||o.oMatchesSelector||o.msMatchesSelector))&&ia(function(a){c.disconnectedMatch=s.call(a,"div"),s.call(a,"[s!='']:x"),r.push("!=",O)}),q=q.length&&new RegExp(q.join("|")),r=r.length&&new RegExp(r.join("|")),b=Z.test(o.compareDocumentPosition),t=b||Z.test(o.contains)?function(a,b){var c=9===a.nodeType?a.documentElement:a,d=b&&b.parentNode;return a===d||!(!d||1!==d.nodeType||!(c.contains?c.contains(d):a.compareDocumentPosition&&16&a.compareDocumentPosition(d)))}:function(a,b){if(b)while(b=b.parentNode)if(b===a)return!0;return!1},B=b?function(a,b){if(a===b)return l=!0,0;var d=!a.compareDocumentPosition-!b.compareDocumentPosition;return d?d:(d=(a.ownerDocument||a)===(b.ownerDocument||b)?a.compareDocumentPosition(b):1,1&d||!c.sortDetached&&b.compareDocumentPosition(a)===d?a===n||a.ownerDocument===v&&t(v,a)?-1:b===n||b.ownerDocument===v&&t(v,b)?1:k?J(k,a)-J(k,b):0:4&d?-1:1)}:function(a,b){if(a===b)return l=!0,0;var c,d=0,e=a.parentNode,f=b.parentNode,g=[a],h=[b];if(!e||!f)return a===n?-1:b===n?1:e?-1:f?1:k?J(k,a)-J(k,b):0;if(e===f)return ka(a,b);c=a;while(c=c.parentNode)g.unshift(c);c=b;while(c=c.parentNode)h.unshift(c);while(g[d]===h[d])d++;return d?ka(g[d],h[d]):g[d]===v?-1:h[d]===v?1:0},n):n},fa.matches=function(a,b){return fa(a,null,null,b)},fa.matchesSelector=function(a,b){if((a.ownerDocument||a)!==n&&m(a),b=b.replace(T,"='$1']"),c.matchesSelector&&p&&!A[b+" "]&&(!r||!r.test(b))&&(!q||!q.test(b)))try{var d=s.call(a,b);if(d||c.disconnectedMatch||a.document&&11!==a.document.nodeType)return d}catch(e){}return fa(b,n,null,[a]).length>0},fa.contains=function(a,b){return(a.ownerDocument||a)!==n&&m(a),t(a,b)},fa.attr=function(a,b){(a.ownerDocument||a)!==n&&m(a);var e=d.attrHandle[b.toLowerCase()],f=e&&D.call(d.attrHandle,b.toLowerCase())?e(a,b,!p):void 0;return void 0!==f?f:c.attributes||!p?a.getAttribute(b):(f=a.getAttributeNode(b))&&f.specified?f.value:null},fa.error=function(a){throw new Error("Syntax error, unrecognized expression: "+a)},fa.uniqueSort=function(a){var b,d=[],e=0,f=0;if(l=!c.detectDuplicates,k=!c.sortStable&&a.slice(0),a.sort(B),l){while(b=a[f++])b===a[f]&&(e=d.push(f));while(e--)a.splice(d[e],1)}return k=null,a},e=fa.getText=function(a){var b,c="",d=0,f=a.nodeType;if(f){if(1===f||9===f||11===f){if("string"==typeof a.textContent)return a.textContent;for(a=a.firstChild;a;a=a.nextSibling)c+=e(a)}else if(3===f||4===f)return a.nodeValue}else while(b=a[d++])c+=e(b);return c},d=fa.selectors={cacheLength:50,createPseudo:ha,match:W,attrHandle:{},find:{},relative:{">":{dir:"parentNode",first:!0}," ":{dir:"parentNode"},"+":{dir:"previousSibling",first:!0},"~":{dir:"previousSibling"}},preFilter:{ATTR:function(a){return a[1]=a[1].replace(ba,ca),a[3]=(a[3]||a[4]||a[5]||"").replace(ba,ca),"~="===a[2]&&(a[3]=" "+a[3]+" "),a.slice(0,4)},CHILD:function(a){return a[1]=a[1].toLowerCase(),"nth"===a[1].slice(0,3)?(a[3]||fa.error(a[0]),a[4]=+(a[4]?a[5]+(a[6]||1):2*("even"===a[3]||"odd"===a[3])),a[5]=+(a[7]+a[8]||"odd"===a[3])):a[3]&&fa.error(a[0]),a},PSEUDO:function(a){var b,c=!a[6]&&a[2];return W.CHILD.test(a[0])?null:(a[3]?a[2]=a[4]||a[5]||"":c&&U.test(c)&&(b=g(c,!0))&&(b=c.indexOf(")",c.length-b)-c.length)&&(a[0]=a[0].slice(0,b),a[2]=c.slice(0,b)),a.slice(0,3))}},filter:{TAG:function(a){var b=a.replace(ba,ca).toLowerCase();return"*"===a?function(){return!0}:function(a){return a.nodeName&&a.nodeName.toLowerCase()===b}},CLASS:function(a){var b=y[a+" "];return b||(b=new RegExp("(^|"+L+")"+a+"("+L+"|$)"))&&y(a,function(a){return b.test("string"==typeof a.className&&a.className||"undefined"!=typeof a.getAttribute&&a.getAttribute("class")||"")})},ATTR:function(a,b,c){return function(d){var e=fa.attr(d,a);return null==e?"!="===b:b?(e+="","="===b?e===c:"!="===b?e!==c:"^="===b?c&&0===e.indexOf(c):"*="===b?c&&e.indexOf(c)>-1:"$="===b?c&&e.slice(-c.length)===c:"~="===b?(" "+e.replace(P," ")+" ").indexOf(c)>-1:"|="===b?e===c||e.slice(0,c.length+1)===c+"-":!1):!0}},CHILD:function(a,b,c,d,e){var f="nth"!==a.slice(0,3),g="last"!==a.slice(-4),h="of-type"===b;return 1===d&&0===e?function(a){return!!a.parentNode}:function(b,c,i){var j,k,l,m,n,o,p=f!==g?"nextSibling":"previousSibling",q=b.parentNode,r=h&&b.nodeName.toLowerCase(),s=!i&&!h,t=!1;if(q){if(f){while(p){m=b;while(m=m[p])if(h?m.nodeName.toLowerCase()===r:1===m.nodeType)return!1;o=p="only"===a&&!o&&"nextSibling"}return!0}if(o=[g?q.firstChild:q.lastChild],g&&s){m=q,l=m[u]||(m[u]={}),k=l[m.uniqueID]||(l[m.uniqueID]={}),j=k[a]||[],n=j[0]===w&&j[1],t=n&&j[2],m=n&&q.childNodes[n];while(m=++n&&m&&m[p]||(t=n=0)||o.pop())if(1===m.nodeType&&++t&&m===b){k[a]=[w,n,t];break}}else if(s&&(m=b,l=m[u]||(m[u]={}),k=l[m.uniqueID]||(l[m.uniqueID]={}),j=k[a]||[],n=j[0]===w&&j[1],t=n),t===!1)while(m=++n&&m&&m[p]||(t=n=0)||o.pop())if((h?m.nodeName.toLowerCase()===r:1===m.nodeType)&&++t&&(s&&(l=m[u]||(m[u]={}),k=l[m.uniqueID]||(l[m.uniqueID]={}),k[a]=[w,t]),m===b))break;return t-=e,t===d||t%d===0&&t/d>=0}}},PSEUDO:function(a,b){var c,e=d.pseudos[a]||d.setFilters[a.toLowerCase()]||fa.error("unsupported pseudo: "+a);return e[u]?e(b):e.length>1?(c=[a,a,"",b],d.setFilters.hasOwnProperty(a.toLowerCase())?ha(function(a,c){var d,f=e(a,b),g=f.length;while(g--)d=J(a,f[g]),a[d]=!(c[d]=f[g])}):function(a){return e(a,0,c)}):e}},pseudos:{not:ha(function(a){var b=[],c=[],d=h(a.replace(Q,"$1"));return d[u]?ha(function(a,b,c,e){var f,g=d(a,null,e,[]),h=a.length;while(h--)(f=g[h])&&(a[h]=!(b[h]=f))}):function(a,e,f){return b[0]=a,d(b,null,f,c),b[0]=null,!c.pop()}}),has:ha(function(a){return function(b){return fa(a,b).length>0}}),contains:ha(function(a){return a=a.replace(ba,ca),function(b){return(b.textContent||b.innerText||e(b)).indexOf(a)>-1}}),lang:ha(function(a){return V.test(a||"")||fa.error("unsupported lang: "+a),a=a.replace(ba,ca).toLowerCase(),function(b){var c;do if(c=p?b.lang:b.getAttribute("xml:lang")||b.getAttribute("lang"))return c=c.toLowerCase(),c===a||0===c.indexOf(a+"-");while((b=b.parentNode)&&1===b.nodeType);return!1}}),target:function(b){var c=a.location&&a.location.hash;return c&&c.slice(1)===b.id},root:function(a){return a===o},focus:function(a){return a===n.activeElement&&(!n.hasFocus||n.hasFocus())&&!!(a.type||a.href||~a.tabIndex)},enabled:function(a){return a.disabled===!1},disabled:function(a){return a.disabled===!0},checked:function(a){var b=a.nodeName.toLowerCase();return"input"===b&&!!a.checked||"option"===b&&!!a.selected},selected:function(a){return a.parentNode&&a.parentNode.selectedIndex,a.selected===!0},empty:function(a){for(a=a.firstChild;a;a=a.nextSibling)if(a.nodeType<6)return!1;return!0},parent:function(a){return!d.pseudos.empty(a)},header:function(a){return Y.test(a.nodeName)},input:function(a){return X.test(a.nodeName)},button:function(a){var b=a.nodeName.toLowerCase();return"input"===b&&"button"===a.type||"button"===b},text:function(a){var b;return"input"===a.nodeName.toLowerCase()&&"text"===a.type&&(null==(b=a.getAttribute("type"))||"text"===b.toLowerCase())},first:na(function(){return[0]}),last:na(function(a,b){return[b-1]}),eq:na(function(a,b,c){return[0>c?c+b:c]}),even:na(function(a,b){for(var c=0;b>c;c+=2)a.push(c);return a}),odd:na(function(a,b){for(var c=1;b>c;c+=2)a.push(c);return a}),lt:na(function(a,b,c){for(var d=0>c?c+b:c;--d>=0;)a.push(d);return a}),gt:na(function(a,b,c){for(var d=0>c?c+b:c;++d<b;)a.push(d);return a})}},d.pseudos.nth=d.pseudos.eq;for(b in{radio:!0,checkbox:!0,file:!0,password:!0,image:!0})d.pseudos[b]=la(b);for(b in{submit:!0,reset:!0})d.pseudos[b]=ma(b);function pa(){}pa.prototype=d.filters=d.pseudos,d.setFilters=new pa,g=fa.tokenize=function(a,b){var c,e,f,g,h,i,j,k=z[a+" "];if(k)return b?0:k.slice(0);h=a,i=[],j=d.preFilter;while(h){c&&!(e=R.exec(h))||(e&&(h=h.slice(e[0].length)||h),i.push(f=[])),c=!1,(e=S.exec(h))&&(c=e.shift(),f.push({value:c,type:e[0].replace(Q," ")}),h=h.slice(c.length));for(g in d.filter)!(e=W[g].exec(h))||j[g]&&!(e=j[g](e))||(c=e.shift(),f.push({value:c,type:g,matches:e}),h=h.slice(c.length));if(!c)break}return b?h.length:h?fa.error(a):z(a,i).slice(0)};function qa(a){for(var b=0,c=a.length,d="";c>b;b++)d+=a[b].value;return d}function ra(a,b,c){var d=b.dir,e=c&&"parentNode"===d,f=x++;return b.first?function(b,c,f){while(b=b[d])if(1===b.nodeType||e)return a(b,c,f)}:function(b,c,g){var h,i,j,k=[w,f];if(g){while(b=b[d])if((1===b.nodeType||e)&&a(b,c,g))return!0}else while(b=b[d])if(1===b.nodeType||e){if(j=b[u]||(b[u]={}),i=j[b.uniqueID]||(j[b.uniqueID]={}),(h=i[d])&&h[0]===w&&h[1]===f)return k[2]=h[2];if(i[d]=k,k[2]=a(b,c,g))return!0}}}function sa(a){return a.length>1?function(b,c,d){var e=a.length;while(e--)if(!a[e](b,c,d))return!1;return!0}:a[0]}function ta(a,b,c){for(var d=0,e=b.length;e>d;d++)fa(a,b[d],c);return c}function ua(a,b,c,d,e){for(var f,g=[],h=0,i=a.length,j=null!=b;i>h;h++)(f=a[h])&&(c&&!c(f,d,e)||(g.push(f),j&&b.push(h)));return g}function va(a,b,c,d,e,f){return d&&!d[u]&&(d=va(d)),e&&!e[u]&&(e=va(e,f)),ha(function(f,g,h,i){var j,k,l,m=[],n=[],o=g.length,p=f||ta(b||"*",h.nodeType?[h]:h,[]),q=!a||!f&&b?p:ua(p,m,a,h,i),r=c?e||(f?a:o||d)?[]:g:q;if(c&&c(q,r,h,i),d){j=ua(r,n),d(j,[],h,i),k=j.length;while(k--)(l=j[k])&&(r[n[k]]=!(q[n[k]]=l))}if(f){if(e||a){if(e){j=[],k=r.length;while(k--)(l=r[k])&&j.push(q[k]=l);e(null,r=[],j,i)}k=r.length;while(k--)(l=r[k])&&(j=e?J(f,l):m[k])>-1&&(f[j]=!(g[j]=l))}}else r=ua(r===g?r.splice(o,r.length):r),e?e(null,g,r,i):H.apply(g,r)})}function wa(a){for(var b,c,e,f=a.length,g=d.relative[a[0].type],h=g||d.relative[" "],i=g?1:0,k=ra(function(a){return a===b},h,!0),l=ra(function(a){return J(b,a)>-1},h,!0),m=[function(a,c,d){var e=!g&&(d||c!==j)||((b=c).nodeType?k(a,c,d):l(a,c,d));return b=null,e}];f>i;i++)if(c=d.relative[a[i].type])m=[ra(sa(m),c)];else{if(c=d.filter[a[i].type].apply(null,a[i].matches),c[u]){for(e=++i;f>e;e++)if(d.relative[a[e].type])break;return va(i>1&&sa(m),i>1&&qa(a.slice(0,i-1).concat({value:" "===a[i-2].type?"*":""})).replace(Q,"$1"),c,e>i&&wa(a.slice(i,e)),f>e&&wa(a=a.slice(e)),f>e&&qa(a))}m.push(c)}return sa(m)}function xa(a,b){var c=b.length>0,e=a.length>0,f=function(f,g,h,i,k){var l,o,q,r=0,s="0",t=f&&[],u=[],v=j,x=f||e&&d.find.TAG("*",k),y=w+=null==v?1:Math.random()||.1,z=x.length;for(k&&(j=g===n||g||k);s!==z&&null!=(l=x[s]);s++){if(e&&l){o=0,g||l.ownerDocument===n||(m(l),h=!p);while(q=a[o++])if(q(l,g||n,h)){i.push(l);break}k&&(w=y)}c&&((l=!q&&l)&&r--,f&&t.push(l))}if(r+=s,c&&s!==r){o=0;while(q=b[o++])q(t,u,g,h);if(f){if(r>0)while(s--)t[s]||u[s]||(u[s]=F.call(i));u=ua(u)}H.apply(i,u),k&&!f&&u.length>0&&r+b.length>1&&fa.uniqueSort(i)}return k&&(w=y,j=v),t};return c?ha(f):f}return h=fa.compile=function(a,b){var c,d=[],e=[],f=A[a+" "];if(!f){b||(b=g(a)),c=b.length;while(c--)f=wa(b[c]),f[u]?d.push(f):e.push(f);f=A(a,xa(e,d)),f.selector=a}return f},i=fa.select=function(a,b,e,f){var i,j,k,l,m,n="function"==typeof a&&a,o=!f&&g(a=n.selector||a);if(e=e||[],1===o.length){if(j=o[0]=o[0].slice(0),j.length>2&&"ID"===(k=j[0]).type&&c.getById&&9===b.nodeType&&p&&d.relative[j[1].type]){if(b=(d.find.ID(k.matches[0].replace(ba,ca),b)||[])[0],!b)return e;n&&(b=b.parentNode),a=a.slice(j.shift().value.length)}i=W.needsContext.test(a)?0:j.length;while(i--){if(k=j[i],d.relative[l=k.type])break;if((m=d.find[l])&&(f=m(k.matches[0].replace(ba,ca),_.test(j[0].type)&&oa(b.parentNode)||b))){if(j.splice(i,1),a=f.length&&qa(j),!a)return H.apply(e,f),e;break}}}return(n||h(a,o))(f,b,!p,e,!b||_.test(a)&&oa(b.parentNode)||b),e},c.sortStable=u.split("").sort(B).join("")===u,c.detectDuplicates=!!l,m(),c.sortDetached=ia(function(a){return 1&a.compareDocumentPosition(n.createElement("div"))}),ia(function(a){return a.innerHTML="<a href='#'></a>","#"===a.firstChild.getAttribute("href")})||ja("type|href|height|width",function(a,b,c){return c?void 0:a.getAttribute(b,"type"===b.toLowerCase()?1:2)}),c.attributes&&ia(function(a){return a.innerHTML="<input/>",a.firstChild.setAttribute("value",""),""===a.firstChild.getAttribute("value")})||ja("value",function(a,b,c){return c||"input"!==a.nodeName.toLowerCase()?void 0:a.defaultValue}),ia(function(a){return null==a.getAttribute("disabled")})||ja(K,function(a,b,c){var d;return c?void 0:a[b]===!0?b.toLowerCase():(d=a.getAttributeNode(b))&&d.specified?d.value:null}),fa}(a);n.find=t,n.expr=t.selectors,n.expr[":"]=n.expr.pseudos,n.uniqueSort=n.unique=t.uniqueSort,n.text=t.getText,n.isXMLDoc=t.isXML,n.contains=t.contains;var u=function(a,b,c){var d=[],e=void 0!==c;while((a=a[b])&&9!==a.nodeType)if(1===a.nodeType){if(e&&n(a).is(c))break;d.push(a)}return d},v=function(a,b){for(var c=[];a;a=a.nextSibling)1===a.nodeType&&a!==b&&c.push(a);return c},w=n.expr.match.needsContext,x=/^<([\w-]+)\s*\/?>(?:<\/\1>|)$/,y=/^.[^:#\[\.,]*$/;function z(a,b,c){if(n.isFunction(b))return n.grep(a,function(a,d){return!!b.call(a,d,a)!==c});if(b.nodeType)return n.grep(a,function(a){return a===b!==c});if("string"==typeof b){if(y.test(b))return n.filter(b,a,c);b=n.filter(b,a)}return n.grep(a,function(a){return n.inArray(a,b)>-1!==c})}n.filter=function(a,b,c){var d=b[0];return c&&(a=":not("+a+")"),1===b.length&&1===d.nodeType?n.find.matchesSelector(d,a)?[d]:[]:n.find.matches(a,n.grep(b,function(a){return 1===a.nodeType}))},n.fn.extend({find:function(a){var b,c=[],d=this,e=d.length;if("string"!=typeof a)return this.pushStack(n(a).filter(function(){for(b=0;e>b;b++)if(n.contains(d[b],this))return!0}));for(b=0;e>b;b++)n.find(a,d[b],c);return c=this.pushStack(e>1?n.unique(c):c),c.selector=this.selector?this.selector+" "+a:a,c},filter:function(a){return this.pushStack(z(this,a||[],!1))},not:function(a){return this.pushStack(z(this,a||[],!0))},is:function(a){return!!z(this,"string"==typeof a&&w.test(a)?n(a):a||[],!1).length}});var A,B=/^(?:\s*(<[\w\W]+>)[^>]*|#([\w-]*))$/,C=n.fn.init=function(a,b,c){var e,f;if(!a)return this;if(c=c||A,"string"==typeof a){if(e="<"===a.charAt(0)&&">"===a.charAt(a.length-1)&&a.length>=3?[null,a,null]:B.exec(a),!e||!e[1]&&b)return!b||b.jquery?(b||c).find(a):this.constructor(b).find(a);if(e[1]){if(b=b instanceof n?b[0]:b,n.merge(this,n.parseHTML(e[1],b&&b.nodeType?b.ownerDocument||b:d,!0)),x.test(e[1])&&n.isPlainObject(b))for(e in b)n.isFunction(this[e])?this[e](b[e]):this.attr(e,b[e]);return this}if(f=d.getElementById(e[2]),f&&f.parentNode){if(f.id!==e[2])return A.find(a);this.length=1,this[0]=f}return this.context=d,this.selector=a,this}return a.nodeType?(this.context=this[0]=a,this.length=1,this):n.isFunction(a)?"undefined"!=typeof c.ready?c.ready(a):a(n):(void 0!==a.selector&&(this.selector=a.selector,this.context=a.context),n.makeArray(a,this))};C.prototype=n.fn,A=n(d);var D=/^(?:parents|prev(?:Until|All))/,E={children:!0,contents:!0,next:!0,prev:!0};n.fn.extend({has:function(a){var b,c=n(a,this),d=c.length;return this.filter(function(){for(b=0;d>b;b++)if(n.contains(this,c[b]))return!0})},closest:function(a,b){for(var c,d=0,e=this.length,f=[],g=w.test(a)||"string"!=typeof a?n(a,b||this.context):0;e>d;d++)for(c=this[d];c&&c!==b;c=c.parentNode)if(c.nodeType<11&&(g?g.index(c)>-1:1===c.nodeType&&n.find.matchesSelector(c,a))){f.push(c);break}return this.pushStack(f.length>1?n.uniqueSort(f):f)},index:function(a){return a?"string"==typeof a?n.inArray(this[0],n(a)):n.inArray(a.jquery?a[0]:a,this):this[0]&&this[0].parentNode?this.first().prevAll().length:-1},add:function(a,b){return this.pushStack(n.uniqueSort(n.merge(this.get(),n(a,b))))},addBack:function(a){return this.add(null==a?this.prevObject:this.prevObject.filter(a))}});function F(a,b){do a=a[b];while(a&&1!==a.nodeType);return a}n.each({parent:function(a){var b=a.parentNode;return b&&11!==b.nodeType?b:null},parents:function(a){return u(a,"parentNode")},parentsUntil:function(a,b,c){return u(a,"parentNode",c)},next:function(a){return F(a,"nextSibling")},prev:function(a){return F(a,"previousSibling")},nextAll:function(a){return u(a,"nextSibling")},prevAll:function(a){return u(a,"previousSibling")},nextUntil:function(a,b,c){return u(a,"nextSibling",c)},prevUntil:function(a,b,c){return u(a,"previousSibling",c)},siblings:function(a){return v((a.parentNode||{}).firstChild,a)},children:function(a){return v(a.firstChild)},contents:function(a){return n.nodeName(a,"iframe")?a.contentDocument||a.contentWindow.document:n.merge([],a.childNodes)}},function(a,b){n.fn[a]=function(c,d){var e=n.map(this,b,c);return"Until"!==a.slice(-5)&&(d=c),d&&"string"==typeof d&&(e=n.filter(d,e)),this.length>1&&(E[a]||(e=n.uniqueSort(e)),D.test(a)&&(e=e.reverse())),this.pushStack(e)}});var G=/\S+/g;function H(a){var b={};return n.each(a.match(G)||[],function(a,c){b[c]=!0}),b}n.Callbacks=function(a){a="string"==typeof a?H(a):n.extend({},a);var b,c,d,e,f=[],g=[],h=-1,i=function(){for(e=a.once,d=b=!0;g.length;h=-1){c=g.shift();while(++h<f.length)f[h].apply(c[0],c[1])===!1&&a.stopOnFalse&&(h=f.length,c=!1)}a.memory||(c=!1),b=!1,e&&(f=c?[]:"")},j={add:function(){return f&&(c&&!b&&(h=f.length-1,g.push(c)),function d(b){n.each(b,function(b,c){n.isFunction(c)?a.unique&&j.has(c)||f.push(c):c&&c.length&&"string"!==n.type(c)&&d(c)})}(arguments),c&&!b&&i()),this},remove:function(){return n.each(arguments,function(a,b){var c;while((c=n.inArray(b,f,c))>-1)f.splice(c,1),h>=c&&h--}),this},has:function(a){return a?n.inArray(a,f)>-1:f.length>0},empty:function(){return f&&(f=[]),this},disable:function(){return e=g=[],f=c="",this},disabled:function(){return!f},lock:function(){return e=!0,c||j.disable(),this},locked:function(){return!!e},fireWith:function(a,c){return e||(c=c||[],c=[a,c.slice?c.slice():c],g.push(c),b||i()),this},fire:function(){return j.fireWith(this,arguments),this},fired:function(){return!!d}};return j},n.extend({Deferred:function(a){var b=[["resolve","done",n.Callbacks("once memory"),"resolved"],["reject","fail",n.Callbacks("once memory"),"rejected"],["notify","progress",n.Callbacks("memory")]],c="pending",d={state:function(){return c},always:function(){return e.done(arguments).fail(arguments),this},then:function(){var a=arguments;return n.Deferred(function(c){n.each(b,function(b,f){var g=n.isFunction(a[b])&&a[b];e[f[1]](function(){var a=g&&g.apply(this,arguments);a&&n.isFunction(a.promise)?a.promise().progress(c.notify).done(c.resolve).fail(c.reject):c[f[0]+"With"](this===d?c.promise():this,g?[a]:arguments)})}),a=null}).promise()},promise:function(a){return null!=a?n.extend(a,d):d}},e={};return d.pipe=d.then,n.each(b,function(a,f){var g=f[2],h=f[3];d[f[1]]=g.add,h&&g.add(function(){c=h},b[1^a][2].disable,b[2][2].lock),e[f[0]]=function(){return e[f[0]+"With"](this===e?d:this,arguments),this},e[f[0]+"With"]=g.fireWith}),d.promise(e),a&&a.call(e,e),e},when:function(a){var b=0,c=e.call(arguments),d=c.length,f=1!==d||a&&n.isFunction(a.promise)?d:0,g=1===f?a:n.Deferred(),h=function(a,b,c){return function(d){b[a]=this,c[a]=arguments.length>1?e.call(arguments):d,c===i?g.notifyWith(b,c):--f||g.resolveWith(b,c)}},i,j,k;if(d>1)for(i=new Array(d),j=new Array(d),k=new Array(d);d>b;b++)c[b]&&n.isFunction(c[b].promise)?c[b].promise().progress(h(b,j,i)).done(h(b,k,c)).fail(g.reject):--f;return f||g.resolveWith(k,c),g.promise()}});var I;n.fn.ready=function(a){return n.ready.promise().done(a),this},n.extend({isReady:!1,readyWait:1,holdReady:function(a){a?n.readyWait++:n.ready(!0)},ready:function(a){(a===!0?--n.readyWait:n.isReady)||(n.isReady=!0,a!==!0&&--n.readyWait>0||(I.resolveWith(d,[n]),n.fn.triggerHandler&&(n(d).triggerHandler("ready"),n(d).off("ready"))))}});function J(){d.addEventListener?(d.removeEventListener("DOMContentLoaded",K),a.removeEventListener("load",K)):(d.detachEvent("onreadystatechange",K),a.detachEvent("onload",K))}function K(){(d.addEventListener||"load"===a.event.type||"complete"===d.readyState)&&(J(),n.ready())}n.ready.promise=function(b){if(!I)if(I=n.Deferred(),"complete"===d.readyState||"loading"!==d.readyState&&!d.documentElement.doScroll)a.setTimeout(n.ready);else if(d.addEventListener)d.addEventListener("DOMContentLoaded",K),a.addEventListener("load",K);else{d.attachEvent("onreadystatechange",K),a.attachEvent("onload",K);var c=!1;try{c=null==a.frameElement&&d.documentElement}catch(e){}c&&c.doScroll&&!function f(){if(!n.isReady){try{c.doScroll("left")}catch(b){return a.setTimeout(f,50)}J(),n.ready()}}()}return I.promise(b)},n.ready.promise();var L;for(L in n(l))break;l.ownFirst="0"===L,l.inlineBlockNeedsLayout=!1,n(function(){var a,b,c,e;c=d.getElementsByTagName("body")[0],c&&c.style&&(b=d.createElement("div"),e=d.createElement("div"),e.style.cssText="position:absolute;border:0;width:0;height:0;top:0;left:-9999px",c.appendChild(e).appendChild(b),"undefined"!=typeof b.style.zoom&&(b.style.cssText="display:inline;margin:0;border:0;padding:1px;width:1px;zoom:1",l.inlineBlockNeedsLayout=a=3===b.offsetWidth,a&&(c.style.zoom=1)),c.removeChild(e))}),function(){var a=d.createElement("div");l.deleteExpando=!0;try{delete a.test}catch(b){l.deleteExpando=!1}a=null}();var M=function(a){var b=n.noData[(a.nodeName+" ").toLowerCase()],c=+a.nodeType||1;return 1!==c&&9!==c?!1:!b||b!==!0&&a.getAttribute("classid")===b},N=/^(?:\{[\w\W]*\}|\[[\w\W]*\])$/,O=/([A-Z])/g;function P(a,b,c){if(void 0===c&&1===a.nodeType){var d="data-"+b.replace(O,"-$1").toLowerCase();if(c=a.getAttribute(d),"string"==typeof c){try{c="true"===c?!0:"false"===c?!1:"null"===c?null:+c+""===c?+c:N.test(c)?n.parseJSON(c):c}catch(e){}n.data(a,b,c)}else c=void 0;
}return c}function Q(a){var b;for(b in a)if(("data"!==b||!n.isEmptyObject(a[b]))&&"toJSON"!==b)return!1;return!0}function R(a,b,d,e){if(M(a)){var f,g,h=n.expando,i=a.nodeType,j=i?n.cache:a,k=i?a[h]:a[h]&&h;if(k&&j[k]&&(e||j[k].data)||void 0!==d||"string"!=typeof b)return k||(k=i?a[h]=c.pop()||n.guid++:h),j[k]||(j[k]=i?{}:{toJSON:n.noop}),"object"!=typeof b&&"function"!=typeof b||(e?j[k]=n.extend(j[k],b):j[k].data=n.extend(j[k].data,b)),g=j[k],e||(g.data||(g.data={}),g=g.data),void 0!==d&&(g[n.camelCase(b)]=d),"string"==typeof b?(f=g[b],null==f&&(f=g[n.camelCase(b)])):f=g,f}}function S(a,b,c){if(M(a)){var d,e,f=a.nodeType,g=f?n.cache:a,h=f?a[n.expando]:n.expando;if(g[h]){if(b&&(d=c?g[h]:g[h].data)){n.isArray(b)?b=b.concat(n.map(b,n.camelCase)):b in d?b=[b]:(b=n.camelCase(b),b=b in d?[b]:b.split(" ")),e=b.length;while(e--)delete d[b[e]];if(c?!Q(d):!n.isEmptyObject(d))return}(c||(delete g[h].data,Q(g[h])))&&(f?n.cleanData([a],!0):l.deleteExpando||g!=g.window?delete g[h]:g[h]=void 0)}}}n.extend({cache:{},noData:{"applet ":!0,"embed ":!0,"object ":"clsid:D27CDB6E-AE6D-11cf-96B8-444553540000"},hasData:function(a){return a=a.nodeType?n.cache[a[n.expando]]:a[n.expando],!!a&&!Q(a)},data:function(a,b,c){return R(a,b,c)},removeData:function(a,b){return S(a,b)},_data:function(a,b,c){return R(a,b,c,!0)},_removeData:function(a,b){return S(a,b,!0)}}),n.fn.extend({data:function(a,b){var c,d,e,f=this[0],g=f&&f.attributes;if(void 0===a){if(this.length&&(e=n.data(f),1===f.nodeType&&!n._data(f,"parsedAttrs"))){c=g.length;while(c--)g[c]&&(d=g[c].name,0===d.indexOf("data-")&&(d=n.camelCase(d.slice(5)),P(f,d,e[d])));n._data(f,"parsedAttrs",!0)}return e}return"object"==typeof a?this.each(function(){n.data(this,a)}):arguments.length>1?this.each(function(){n.data(this,a,b)}):f?P(f,a,n.data(f,a)):void 0},removeData:function(a){return this.each(function(){n.removeData(this,a)})}}),n.extend({queue:function(a,b,c){var d;return a?(b=(b||"fx")+"queue",d=n._data(a,b),c&&(!d||n.isArray(c)?d=n._data(a,b,n.makeArray(c)):d.push(c)),d||[]):void 0},dequeue:function(a,b){b=b||"fx";var c=n.queue(a,b),d=c.length,e=c.shift(),f=n._queueHooks(a,b),g=function(){n.dequeue(a,b)};"inprogress"===e&&(e=c.shift(),d--),e&&("fx"===b&&c.unshift("inprogress"),delete f.stop,e.call(a,g,f)),!d&&f&&f.empty.fire()},_queueHooks:function(a,b){var c=b+"queueHooks";return n._data(a,c)||n._data(a,c,{empty:n.Callbacks("once memory").add(function(){n._removeData(a,b+"queue"),n._removeData(a,c)})})}}),n.fn.extend({queue:function(a,b){var c=2;return"string"!=typeof a&&(b=a,a="fx",c--),arguments.length<c?n.queue(this[0],a):void 0===b?this:this.each(function(){var c=n.queue(this,a,b);n._queueHooks(this,a),"fx"===a&&"inprogress"!==c[0]&&n.dequeue(this,a)})},dequeue:function(a){return this.each(function(){n.dequeue(this,a)})},clearQueue:function(a){return this.queue(a||"fx",[])},promise:function(a,b){var c,d=1,e=n.Deferred(),f=this,g=this.length,h=function(){--d||e.resolveWith(f,[f])};"string"!=typeof a&&(b=a,a=void 0),a=a||"fx";while(g--)c=n._data(f[g],a+"queueHooks"),c&&c.empty&&(d++,c.empty.add(h));return h(),e.promise(b)}}),function(){var a;l.shrinkWrapBlocks=function(){if(null!=a)return a;a=!1;var b,c,e;return c=d.getElementsByTagName("body")[0],c&&c.style?(b=d.createElement("div"),e=d.createElement("div"),e.style.cssText="position:absolute;border:0;width:0;height:0;top:0;left:-9999px",c.appendChild(e).appendChild(b),"undefined"!=typeof b.style.zoom&&(b.style.cssText="-webkit-box-sizing:content-box;-moz-box-sizing:content-box;box-sizing:content-box;display:block;margin:0;border:0;padding:1px;width:1px;zoom:1",b.appendChild(d.createElement("div")).style.width="5px",a=3!==b.offsetWidth),c.removeChild(e),a):void 0}}();var T=/[+-]?(?:\d*\.|)\d+(?:[eE][+-]?\d+|)/.source,U=new RegExp("^(?:([+-])=|)("+T+")([a-z%]*)$","i"),V=["Top","Right","Bottom","Left"],W=function(a,b){return a=b||a,"none"===n.css(a,"display")||!n.contains(a.ownerDocument,a)};function X(a,b,c,d){var e,f=1,g=20,h=d?function(){return d.cur()}:function(){return n.css(a,b,"")},i=h(),j=c&&c[3]||(n.cssNumber[b]?"":"px"),k=(n.cssNumber[b]||"px"!==j&&+i)&&U.exec(n.css(a,b));if(k&&k[3]!==j){j=j||k[3],c=c||[],k=+i||1;do f=f||".5",k/=f,n.style(a,b,k+j);while(f!==(f=h()/i)&&1!==f&&--g)}return c&&(k=+k||+i||0,e=c[1]?k+(c[1]+1)*c[2]:+c[2],d&&(d.unit=j,d.start=k,d.end=e)),e}var Y=function(a,b,c,d,e,f,g){var h=0,i=a.length,j=null==c;if("object"===n.type(c)){e=!0;for(h in c)Y(a,b,h,c[h],!0,f,g)}else if(void 0!==d&&(e=!0,n.isFunction(d)||(g=!0),j&&(g?(b.call(a,d),b=null):(j=b,b=function(a,b,c){return j.call(n(a),c)})),b))for(;i>h;h++)b(a[h],c,g?d:d.call(a[h],h,b(a[h],c)));return e?a:j?b.call(a):i?b(a[0],c):f},Z=/^(?:checkbox|radio)$/i,$=/<([\w:-]+)/,_=/^$|\/(?:java|ecma)script/i,aa=/^\s+/,ba="abbr|article|aside|audio|bdi|canvas|data|datalist|details|dialog|figcaption|figure|footer|header|hgroup|main|mark|meter|nav|output|picture|progress|section|summary|template|time|video";function ca(a){var b=ba.split("|"),c=a.createDocumentFragment();if(c.createElement)while(b.length)c.createElement(b.pop());return c}!function(){var a=d.createElement("div"),b=d.createDocumentFragment(),c=d.createElement("input");a.innerHTML="  <link/><table></table><a href='/a'>a</a><input type='checkbox'/>",l.leadingWhitespace=3===a.firstChild.nodeType,l.tbody=!a.getElementsByTagName("tbody").length,l.htmlSerialize=!!a.getElementsByTagName("link").length,l.html5Clone="<:nav></:nav>"!==d.createElement("nav").cloneNode(!0).outerHTML,c.type="checkbox",c.checked=!0,b.appendChild(c),l.appendChecked=c.checked,a.innerHTML="<textarea>x</textarea>",l.noCloneChecked=!!a.cloneNode(!0).lastChild.defaultValue,b.appendChild(a),c=d.createElement("input"),c.setAttribute("type","radio"),c.setAttribute("checked","checked"),c.setAttribute("name","t"),a.appendChild(c),l.checkClone=a.cloneNode(!0).cloneNode(!0).lastChild.checked,l.noCloneEvent=!!a.addEventListener,a[n.expando]=1,l.attributes=!a.getAttribute(n.expando)}();var da={option:[1,"<select multiple='multiple'>","</select>"],legend:[1,"<fieldset>","</fieldset>"],area:[1,"<map>","</map>"],param:[1,"<object>","</object>"],thead:[1,"<table>","</table>"],tr:[2,"<table><tbody>","</tbody></table>"],col:[2,"<table><tbody></tbody><colgroup>","</colgroup></table>"],td:[3,"<table><tbody><tr>","</tr></tbody></table>"],_default:l.htmlSerialize?[0,"",""]:[1,"X<div>","</div>"]};da.optgroup=da.option,da.tbody=da.tfoot=da.colgroup=da.caption=da.thead,da.th=da.td;function ea(a,b){var c,d,e=0,f="undefined"!=typeof a.getElementsByTagName?a.getElementsByTagName(b||"*"):"undefined"!=typeof a.querySelectorAll?a.querySelectorAll(b||"*"):void 0;if(!f)for(f=[],c=a.childNodes||a;null!=(d=c[e]);e++)!b||n.nodeName(d,b)?f.push(d):n.merge(f,ea(d,b));return void 0===b||b&&n.nodeName(a,b)?n.merge([a],f):f}function fa(a,b){for(var c,d=0;null!=(c=a[d]);d++)n._data(c,"globalEval",!b||n._data(b[d],"globalEval"))}var ga=/<|&#?\w+;/,ha=/<tbody/i;function ia(a){Z.test(a.type)&&(a.defaultChecked=a.checked)}function ja(a,b,c,d,e){for(var f,g,h,i,j,k,m,o=a.length,p=ca(b),q=[],r=0;o>r;r++)if(g=a[r],g||0===g)if("object"===n.type(g))n.merge(q,g.nodeType?[g]:g);else if(ga.test(g)){i=i||p.appendChild(b.createElement("div")),j=($.exec(g)||["",""])[1].toLowerCase(),m=da[j]||da._default,i.innerHTML=m[1]+n.htmlPrefilter(g)+m[2],f=m[0];while(f--)i=i.lastChild;if(!l.leadingWhitespace&&aa.test(g)&&q.push(b.createTextNode(aa.exec(g)[0])),!l.tbody){g="table"!==j||ha.test(g)?"<table>"!==m[1]||ha.test(g)?0:i:i.firstChild,f=g&&g.childNodes.length;while(f--)n.nodeName(k=g.childNodes[f],"tbody")&&!k.childNodes.length&&g.removeChild(k)}n.merge(q,i.childNodes),i.textContent="";while(i.firstChild)i.removeChild(i.firstChild);i=p.lastChild}else q.push(b.createTextNode(g));i&&p.removeChild(i),l.appendChecked||n.grep(ea(q,"input"),ia),r=0;while(g=q[r++])if(d&&n.inArray(g,d)>-1)e&&e.push(g);else if(h=n.contains(g.ownerDocument,g),i=ea(p.appendChild(g),"script"),h&&fa(i),c){f=0;while(g=i[f++])_.test(g.type||"")&&c.push(g)}return i=null,p}!function(){var b,c,e=d.createElement("div");for(b in{submit:!0,change:!0,focusin:!0})c="on"+b,(l[b]=c in a)||(e.setAttribute(c,"t"),l[b]=e.attributes[c].expando===!1);e=null}();var ka=/^(?:input|select|textarea)$/i,la=/^key/,ma=/^(?:mouse|pointer|contextmenu|drag|drop)|click/,na=/^(?:focusinfocus|focusoutblur)$/,oa=/^([^.]*)(?:\.(.+)|)/;function pa(){return!0}function qa(){return!1}function ra(){try{return d.activeElement}catch(a){}}function sa(a,b,c,d,e,f){var g,h;if("object"==typeof b){"string"!=typeof c&&(d=d||c,c=void 0);for(h in b)sa(a,h,c,d,b[h],f);return a}if(null==d&&null==e?(e=c,d=c=void 0):null==e&&("string"==typeof c?(e=d,d=void 0):(e=d,d=c,c=void 0)),e===!1)e=qa;else if(!e)return a;return 1===f&&(g=e,e=function(a){return n().off(a),g.apply(this,arguments)},e.guid=g.guid||(g.guid=n.guid++)),a.each(function(){n.event.add(this,b,e,d,c)})}n.event={global:{},add:function(a,b,c,d,e){var f,g,h,i,j,k,l,m,o,p,q,r=n._data(a);if(r){c.handler&&(i=c,c=i.handler,e=i.selector),c.guid||(c.guid=n.guid++),(g=r.events)||(g=r.events={}),(k=r.handle)||(k=r.handle=function(a){return"undefined"==typeof n||a&&n.event.triggered===a.type?void 0:n.event.dispatch.apply(k.elem,arguments)},k.elem=a),b=(b||"").match(G)||[""],h=b.length;while(h--)f=oa.exec(b[h])||[],o=q=f[1],p=(f[2]||"").split(".").sort(),o&&(j=n.event.special[o]||{},o=(e?j.delegateType:j.bindType)||o,j=n.event.special[o]||{},l=n.extend({type:o,origType:q,data:d,handler:c,guid:c.guid,selector:e,needsContext:e&&n.expr.match.needsContext.test(e),namespace:p.join(".")},i),(m=g[o])||(m=g[o]=[],m.delegateCount=0,j.setup&&j.setup.call(a,d,p,k)!==!1||(a.addEventListener?a.addEventListener(o,k,!1):a.attachEvent&&a.attachEvent("on"+o,k))),j.add&&(j.add.call(a,l),l.handler.guid||(l.handler.guid=c.guid)),e?m.splice(m.delegateCount++,0,l):m.push(l),n.event.global[o]=!0);a=null}},remove:function(a,b,c,d,e){var f,g,h,i,j,k,l,m,o,p,q,r=n.hasData(a)&&n._data(a);if(r&&(k=r.events)){b=(b||"").match(G)||[""],j=b.length;while(j--)if(h=oa.exec(b[j])||[],o=q=h[1],p=(h[2]||"").split(".").sort(),o){l=n.event.special[o]||{},o=(d?l.delegateType:l.bindType)||o,m=k[o]||[],h=h[2]&&new RegExp("(^|\\.)"+p.join("\\.(?:.*\\.|)")+"(\\.|$)"),i=f=m.length;while(f--)g=m[f],!e&&q!==g.origType||c&&c.guid!==g.guid||h&&!h.test(g.namespace)||d&&d!==g.selector&&("**"!==d||!g.selector)||(m.splice(f,1),g.selector&&m.delegateCount--,l.remove&&l.remove.call(a,g));i&&!m.length&&(l.teardown&&l.teardown.call(a,p,r.handle)!==!1||n.removeEvent(a,o,r.handle),delete k[o])}else for(o in k)n.event.remove(a,o+b[j],c,d,!0);n.isEmptyObject(k)&&(delete r.handle,n._removeData(a,"events"))}},trigger:function(b,c,e,f){var g,h,i,j,l,m,o,p=[e||d],q=k.call(b,"type")?b.type:b,r=k.call(b,"namespace")?b.namespace.split("."):[];if(i=m=e=e||d,3!==e.nodeType&&8!==e.nodeType&&!na.test(q+n.event.triggered)&&(q.indexOf(".")>-1&&(r=q.split("."),q=r.shift(),r.sort()),h=q.indexOf(":")<0&&"on"+q,b=b[n.expando]?b:new n.Event(q,"object"==typeof b&&b),b.isTrigger=f?2:3,b.namespace=r.join("."),b.rnamespace=b.namespace?new RegExp("(^|\\.)"+r.join("\\.(?:.*\\.|)")+"(\\.|$)"):null,b.result=void 0,b.target||(b.target=e),c=null==c?[b]:n.makeArray(c,[b]),l=n.event.special[q]||{},f||!l.trigger||l.trigger.apply(e,c)!==!1)){if(!f&&!l.noBubble&&!n.isWindow(e)){for(j=l.delegateType||q,na.test(j+q)||(i=i.parentNode);i;i=i.parentNode)p.push(i),m=i;m===(e.ownerDocument||d)&&p.push(m.defaultView||m.parentWindow||a)}o=0;while((i=p[o++])&&!b.isPropagationStopped())b.type=o>1?j:l.bindType||q,g=(n._data(i,"events")||{})[b.type]&&n._data(i,"handle"),g&&g.apply(i,c),g=h&&i[h],g&&g.apply&&M(i)&&(b.result=g.apply(i,c),b.result===!1&&b.preventDefault());if(b.type=q,!f&&!b.isDefaultPrevented()&&(!l._default||l._default.apply(p.pop(),c)===!1)&&M(e)&&h&&e[q]&&!n.isWindow(e)){m=e[h],m&&(e[h]=null),n.event.triggered=q;try{e[q]()}catch(s){}n.event.triggered=void 0,m&&(e[h]=m)}return b.result}},dispatch:function(a){a=n.event.fix(a);var b,c,d,f,g,h=[],i=e.call(arguments),j=(n._data(this,"events")||{})[a.type]||[],k=n.event.special[a.type]||{};if(i[0]=a,a.delegateTarget=this,!k.preDispatch||k.preDispatch.call(this,a)!==!1){h=n.event.handlers.call(this,a,j),b=0;while((f=h[b++])&&!a.isPropagationStopped()){a.currentTarget=f.elem,c=0;while((g=f.handlers[c++])&&!a.isImmediatePropagationStopped())a.rnamespace&&!a.rnamespace.test(g.namespace)||(a.handleObj=g,a.data=g.data,d=((n.event.special[g.origType]||{}).handle||g.handler).apply(f.elem,i),void 0!==d&&(a.result=d)===!1&&(a.preventDefault(),a.stopPropagation()))}return k.postDispatch&&k.postDispatch.call(this,a),a.result}},handlers:function(a,b){var c,d,e,f,g=[],h=b.delegateCount,i=a.target;if(h&&i.nodeType&&("click"!==a.type||isNaN(a.button)||a.button<1))for(;i!=this;i=i.parentNode||this)if(1===i.nodeType&&(i.disabled!==!0||"click"!==a.type)){for(d=[],c=0;h>c;c++)f=b[c],e=f.selector+" ",void 0===d[e]&&(d[e]=f.needsContext?n(e,this).index(i)>-1:n.find(e,this,null,[i]).length),d[e]&&d.push(f);d.length&&g.push({elem:i,handlers:d})}return h<b.length&&g.push({elem:this,handlers:b.slice(h)}),g},fix:function(a){if(a[n.expando])return a;var b,c,e,f=a.type,g=a,h=this.fixHooks[f];h||(this.fixHooks[f]=h=ma.test(f)?this.mouseHooks:la.test(f)?this.keyHooks:{}),e=h.props?this.props.concat(h.props):this.props,a=new n.Event(g),b=e.length;while(b--)c=e[b],a[c]=g[c];return a.target||(a.target=g.srcElement||d),3===a.target.nodeType&&(a.target=a.target.parentNode),a.metaKey=!!a.metaKey,h.filter?h.filter(a,g):a},props:"altKey bubbles cancelable ctrlKey currentTarget detail eventPhase metaKey relatedTarget shiftKey target timeStamp view which".split(" "),fixHooks:{},keyHooks:{props:"char charCode key keyCode".split(" "),filter:function(a,b){return null==a.which&&(a.which=null!=b.charCode?b.charCode:b.keyCode),a}},mouseHooks:{props:"button buttons clientX clientY fromElement offsetX offsetY pageX pageY screenX screenY toElement".split(" "),filter:function(a,b){var c,e,f,g=b.button,h=b.fromElement;return null==a.pageX&&null!=b.clientX&&(e=a.target.ownerDocument||d,f=e.documentElement,c=e.body,a.pageX=b.clientX+(f&&f.scrollLeft||c&&c.scrollLeft||0)-(f&&f.clientLeft||c&&c.clientLeft||0),a.pageY=b.clientY+(f&&f.scrollTop||c&&c.scrollTop||0)-(f&&f.clientTop||c&&c.clientTop||0)),!a.relatedTarget&&h&&(a.relatedTarget=h===a.target?b.toElement:h),a.which||void 0===g||(a.which=1&g?1:2&g?3:4&g?2:0),a}},special:{load:{noBubble:!0},focus:{trigger:function(){if(this!==ra()&&this.focus)try{return this.focus(),!1}catch(a){}},delegateType:"focusin"},blur:{trigger:function(){return this===ra()&&this.blur?(this.blur(),!1):void 0},delegateType:"focusout"},click:{trigger:function(){return n.nodeName(this,"input")&&"checkbox"===this.type&&this.click?(this.click(),!1):void 0},_default:function(a){return n.nodeName(a.target,"a")}},beforeunload:{postDispatch:function(a){void 0!==a.result&&a.originalEvent&&(a.originalEvent.returnValue=a.result)}}},simulate:function(a,b,c){var d=n.extend(new n.Event,c,{type:a,isSimulated:!0});n.event.trigger(d,null,b),d.isDefaultPrevented()&&c.preventDefault()}},n.removeEvent=d.removeEventListener?function(a,b,c){a.removeEventListener&&a.removeEventListener(b,c)}:function(a,b,c){var d="on"+b;a.detachEvent&&("undefined"==typeof a[d]&&(a[d]=null),a.detachEvent(d,c))},n.Event=function(a,b){return this instanceof n.Event?(a&&a.type?(this.originalEvent=a,this.type=a.type,this.isDefaultPrevented=a.defaultPrevented||void 0===a.defaultPrevented&&a.returnValue===!1?pa:qa):this.type=a,b&&n.extend(this,b),this.timeStamp=a&&a.timeStamp||n.now(),void(this[n.expando]=!0)):new n.Event(a,b)},n.Event.prototype={constructor:n.Event,isDefaultPrevented:qa,isPropagationStopped:qa,isImmediatePropagationStopped:qa,preventDefault:function(){var a=this.originalEvent;this.isDefaultPrevented=pa,a&&(a.preventDefault?a.preventDefault():a.returnValue=!1)},stopPropagation:function(){var a=this.originalEvent;this.isPropagationStopped=pa,a&&!this.isSimulated&&(a.stopPropagation&&a.stopPropagation(),a.cancelBubble=!0)},stopImmediatePropagation:function(){var a=this.originalEvent;this.isImmediatePropagationStopped=pa,a&&a.stopImmediatePropagation&&a.stopImmediatePropagation(),this.stopPropagation()}},n.each({mouseenter:"mouseover",mouseleave:"mouseout",pointerenter:"pointerover",pointerleave:"pointerout"},function(a,b){n.event.special[a]={delegateType:b,bindType:b,handle:function(a){var c,d=this,e=a.relatedTarget,f=a.handleObj;return e&&(e===d||n.contains(d,e))||(a.type=f.origType,c=f.handler.apply(this,arguments),a.type=b),c}}}),l.submit||(n.event.special.submit={setup:function(){return n.nodeName(this,"form")?!1:void n.event.add(this,"click._submit keypress._submit",function(a){var b=a.target,c=n.nodeName(b,"input")||n.nodeName(b,"button")?n.prop(b,"form"):void 0;c&&!n._data(c,"submit")&&(n.event.add(c,"submit._submit",function(a){a._submitBubble=!0}),n._data(c,"submit",!0))})},postDispatch:function(a){a._submitBubble&&(delete a._submitBubble,this.parentNode&&!a.isTrigger&&n.event.simulate("submit",this.parentNode,a))},teardown:function(){return n.nodeName(this,"form")?!1:void n.event.remove(this,"._submit")}}),l.change||(n.event.special.change={setup:function(){return ka.test(this.nodeName)?("checkbox"!==this.type&&"radio"!==this.type||(n.event.add(this,"propertychange._change",function(a){"checked"===a.originalEvent.propertyName&&(this._justChanged=!0)}),n.event.add(this,"click._change",function(a){this._justChanged&&!a.isTrigger&&(this._justChanged=!1),n.event.simulate("change",this,a)})),!1):void n.event.add(this,"beforeactivate._change",function(a){var b=a.target;ka.test(b.nodeName)&&!n._data(b,"change")&&(n.event.add(b,"change._change",function(a){!this.parentNode||a.isSimulated||a.isTrigger||n.event.simulate("change",this.parentNode,a)}),n._data(b,"change",!0))})},handle:function(a){var b=a.target;return this!==b||a.isSimulated||a.isTrigger||"radio"!==b.type&&"checkbox"!==b.type?a.handleObj.handler.apply(this,arguments):void 0},teardown:function(){return n.event.remove(this,"._change"),!ka.test(this.nodeName)}}),l.focusin||n.each({focus:"focusin",blur:"focusout"},function(a,b){var c=function(a){n.event.simulate(b,a.target,n.event.fix(a))};n.event.special[b]={setup:function(){var d=this.ownerDocument||this,e=n._data(d,b);e||d.addEventListener(a,c,!0),n._data(d,b,(e||0)+1)},teardown:function(){var d=this.ownerDocument||this,e=n._data(d,b)-1;e?n._data(d,b,e):(d.removeEventListener(a,c,!0),n._removeData(d,b))}}}),n.fn.extend({on:function(a,b,c,d){return sa(this,a,b,c,d)},one:function(a,b,c,d){return sa(this,a,b,c,d,1)},off:function(a,b,c){var d,e;if(a&&a.preventDefault&&a.handleObj)return d=a.handleObj,n(a.delegateTarget).off(d.namespace?d.origType+"."+d.namespace:d.origType,d.selector,d.handler),this;if("object"==typeof a){for(e in a)this.off(e,b,a[e]);return this}return b!==!1&&"function"!=typeof b||(c=b,b=void 0),c===!1&&(c=qa),this.each(function(){n.event.remove(this,a,c,b)})},trigger:function(a,b){return this.each(function(){n.event.trigger(a,b,this)})},triggerHandler:function(a,b){var c=this[0];return c?n.event.trigger(a,b,c,!0):void 0}});var ta=/ jQuery\d+="(?:null|\d+)"/g,ua=new RegExp("<(?:"+ba+")[\\s/>]","i"),va=/<(?!area|br|col|embed|hr|img|input|link|meta|param)(([\w:-]+)[^>]*)\/>/gi,wa=/<script|<style|<link/i,xa=/checked\s*(?:[^=]|=\s*.checked.)/i,ya=/^true\/(.*)/,za=/^\s*<!(?:\[CDATA\[|--)|(?:\]\]|--)>\s*$/g,Aa=ca(d),Ba=Aa.appendChild(d.createElement("div"));function Ca(a,b){return n.nodeName(a,"table")&&n.nodeName(11!==b.nodeType?b:b.firstChild,"tr")?a.getElementsByTagName("tbody")[0]||a.appendChild(a.ownerDocument.createElement("tbody")):a}function Da(a){return a.type=(null!==n.find.attr(a,"type"))+"/"+a.type,a}function Ea(a){var b=ya.exec(a.type);return b?a.type=b[1]:a.removeAttribute("type"),a}function Fa(a,b){if(1===b.nodeType&&n.hasData(a)){var c,d,e,f=n._data(a),g=n._data(b,f),h=f.events;if(h){delete g.handle,g.events={};for(c in h)for(d=0,e=h[c].length;e>d;d++)n.event.add(b,c,h[c][d])}g.data&&(g.data=n.extend({},g.data))}}function Ga(a,b){var c,d,e;if(1===b.nodeType){if(c=b.nodeName.toLowerCase(),!l.noCloneEvent&&b[n.expando]){e=n._data(b);for(d in e.events)n.removeEvent(b,d,e.handle);b.removeAttribute(n.expando)}"script"===c&&b.text!==a.text?(Da(b).text=a.text,Ea(b)):"object"===c?(b.parentNode&&(b.outerHTML=a.outerHTML),l.html5Clone&&a.innerHTML&&!n.trim(b.innerHTML)&&(b.innerHTML=a.innerHTML)):"input"===c&&Z.test(a.type)?(b.defaultChecked=b.checked=a.checked,b.value!==a.value&&(b.value=a.value)):"option"===c?b.defaultSelected=b.selected=a.defaultSelected:"input"!==c&&"textarea"!==c||(b.defaultValue=a.defaultValue)}}function Ha(a,b,c,d){b=f.apply([],b);var e,g,h,i,j,k,m=0,o=a.length,p=o-1,q=b[0],r=n.isFunction(q);if(r||o>1&&"string"==typeof q&&!l.checkClone&&xa.test(q))return a.each(function(e){var f=a.eq(e);r&&(b[0]=q.call(this,e,f.html())),Ha(f,b,c,d)});if(o&&(k=ja(b,a[0].ownerDocument,!1,a,d),e=k.firstChild,1===k.childNodes.length&&(k=e),e||d)){for(i=n.map(ea(k,"script"),Da),h=i.length;o>m;m++)g=k,m!==p&&(g=n.clone(g,!0,!0),h&&n.merge(i,ea(g,"script"))),c.call(a[m],g,m);if(h)for(j=i[i.length-1].ownerDocument,n.map(i,Ea),m=0;h>m;m++)g=i[m],_.test(g.type||"")&&!n._data(g,"globalEval")&&n.contains(j,g)&&(g.src?n._evalUrl&&n._evalUrl(g.src):n.globalEval((g.text||g.textContent||g.innerHTML||"").replace(za,"")));k=e=null}return a}function Ia(a,b,c){for(var d,e=b?n.filter(b,a):a,f=0;null!=(d=e[f]);f++)c||1!==d.nodeType||n.cleanData(ea(d)),d.parentNode&&(c&&n.contains(d.ownerDocument,d)&&fa(ea(d,"script")),d.parentNode.removeChild(d));return a}n.extend({htmlPrefilter:function(a){return a.replace(va,"<$1></$2>")},clone:function(a,b,c){var d,e,f,g,h,i=n.contains(a.ownerDocument,a);if(l.html5Clone||n.isXMLDoc(a)||!ua.test("<"+a.nodeName+">")?f=a.cloneNode(!0):(Ba.innerHTML=a.outerHTML,Ba.removeChild(f=Ba.firstChild)),!(l.noCloneEvent&&l.noCloneChecked||1!==a.nodeType&&11!==a.nodeType||n.isXMLDoc(a)))for(d=ea(f),h=ea(a),g=0;null!=(e=h[g]);++g)d[g]&&Ga(e,d[g]);if(b)if(c)for(h=h||ea(a),d=d||ea(f),g=0;null!=(e=h[g]);g++)Fa(e,d[g]);else Fa(a,f);return d=ea(f,"script"),d.length>0&&fa(d,!i&&ea(a,"script")),d=h=e=null,f},cleanData:function(a,b){for(var d,e,f,g,h=0,i=n.expando,j=n.cache,k=l.attributes,m=n.event.special;null!=(d=a[h]);h++)if((b||M(d))&&(f=d[i],g=f&&j[f])){if(g.events)for(e in g.events)m[e]?n.event.remove(d,e):n.removeEvent(d,e,g.handle);j[f]&&(delete j[f],k||"undefined"==typeof d.removeAttribute?d[i]=void 0:d.removeAttribute(i),c.push(f))}}}),n.fn.extend({domManip:Ha,detach:function(a){return Ia(this,a,!0)},remove:function(a){return Ia(this,a)},text:function(a){return Y(this,function(a){return void 0===a?n.text(this):this.empty().append((this[0]&&this[0].ownerDocument||d).createTextNode(a))},null,a,arguments.length)},append:function(){return Ha(this,arguments,function(a){if(1===this.nodeType||11===this.nodeType||9===this.nodeType){var b=Ca(this,a);b.appendChild(a)}})},prepend:function(){return Ha(this,arguments,function(a){if(1===this.nodeType||11===this.nodeType||9===this.nodeType){var b=Ca(this,a);b.insertBefore(a,b.firstChild)}})},before:function(){return Ha(this,arguments,function(a){this.parentNode&&this.parentNode.insertBefore(a,this)})},after:function(){return Ha(this,arguments,function(a){this.parentNode&&this.parentNode.insertBefore(a,this.nextSibling)})},empty:function(){for(var a,b=0;null!=(a=this[b]);b++){1===a.nodeType&&n.cleanData(ea(a,!1));while(a.firstChild)a.removeChild(a.firstChild);a.options&&n.nodeName(a,"select")&&(a.options.length=0)}return this},clone:function(a,b){return a=null==a?!1:a,b=null==b?a:b,this.map(function(){return n.clone(this,a,b)})},html:function(a){return Y(this,function(a){var b=this[0]||{},c=0,d=this.length;if(void 0===a)return 1===b.nodeType?b.innerHTML.replace(ta,""):void 0;if("string"==typeof a&&!wa.test(a)&&(l.htmlSerialize||!ua.test(a))&&(l.leadingWhitespace||!aa.test(a))&&!da[($.exec(a)||["",""])[1].toLowerCase()]){a=n.htmlPrefilter(a);try{for(;d>c;c++)b=this[c]||{},1===b.nodeType&&(n.cleanData(ea(b,!1)),b.innerHTML=a);b=0}catch(e){}}b&&this.empty().append(a)},null,a,arguments.length)},replaceWith:function(){var a=[];return Ha(this,arguments,function(b){var c=this.parentNode;n.inArray(this,a)<0&&(n.cleanData(ea(this)),c&&c.replaceChild(b,this))},a)}}),n.each({appendTo:"append",prependTo:"prepend",insertBefore:"before",insertAfter:"after",replaceAll:"replaceWith"},function(a,b){n.fn[a]=function(a){for(var c,d=0,e=[],f=n(a),h=f.length-1;h>=d;d++)c=d===h?this:this.clone(!0),n(f[d])[b](c),g.apply(e,c.get());return this.pushStack(e)}});var Ja,Ka={HTML:"block",BODY:"block"};function La(a,b){var c=n(b.createElement(a)).appendTo(b.body),d=n.css(c[0],"display");return c.detach(),d}function Ma(a){var b=d,c=Ka[a];return c||(c=La(a,b),"none"!==c&&c||(Ja=(Ja||n("<iframe frameborder='0' width='0' height='0'/>")).appendTo(b.documentElement),b=(Ja[0].contentWindow||Ja[0].contentDocument).document,b.write(),b.close(),c=La(a,b),Ja.detach()),Ka[a]=c),c}var Na=/^margin/,Oa=new RegExp("^("+T+")(?!px)[a-z%]+$","i"),Pa=function(a,b,c,d){var e,f,g={};for(f in b)g[f]=a.style[f],a.style[f]=b[f];e=c.apply(a,d||[]);for(f in b)a.style[f]=g[f];return e},Qa=d.documentElement;!function(){var b,c,e,f,g,h,i=d.createElement("div"),j=d.createElement("div");if(j.style){j.style.cssText="float:left;opacity:.5",l.opacity="0.5"===j.style.opacity,l.cssFloat=!!j.style.cssFloat,j.style.backgroundClip="content-box",j.cloneNode(!0).style.backgroundClip="",l.clearCloneStyle="content-box"===j.style.backgroundClip,i=d.createElement("div"),i.style.cssText="border:0;width:8px;height:0;top:0;left:-9999px;padding:0;margin-top:1px;position:absolute",j.innerHTML="",i.appendChild(j),l.boxSizing=""===j.style.boxSizing||""===j.style.MozBoxSizing||""===j.style.WebkitBoxSizing,n.extend(l,{reliableHiddenOffsets:function(){return null==b&&k(),f},boxSizingReliable:function(){return null==b&&k(),e},pixelMarginRight:function(){return null==b&&k(),c},pixelPosition:function(){return null==b&&k(),b},reliableMarginRight:function(){return null==b&&k(),g},reliableMarginLeft:function(){return null==b&&k(),h}});function k(){var k,l,m=d.documentElement;m.appendChild(i),j.style.cssText="-webkit-box-sizing:border-box;box-sizing:border-box;position:relative;display:block;margin:auto;border:1px;padding:1px;top:1%;width:50%",b=e=h=!1,c=g=!0,a.getComputedStyle&&(l=a.getComputedStyle(j),b="1%"!==(l||{}).top,h="2px"===(l||{}).marginLeft,e="4px"===(l||{width:"4px"}).width,j.style.marginRight="50%",c="4px"===(l||{marginRight:"4px"}).marginRight,k=j.appendChild(d.createElement("div")),k.style.cssText=j.style.cssText="-webkit-box-sizing:content-box;-moz-box-sizing:content-box;box-sizing:content-box;display:block;margin:0;border:0;padding:0",k.style.marginRight=k.style.width="0",j.style.width="1px",g=!parseFloat((a.getComputedStyle(k)||{}).marginRight),j.removeChild(k)),j.style.display="none",f=0===j.getClientRects().length,f&&(j.style.display="",j.innerHTML="<table><tr><td></td><td>t</td></tr></table>",j.childNodes[0].style.borderCollapse="separate",k=j.getElementsByTagName("td"),k[0].style.cssText="margin:0;border:0;padding:0;display:none",f=0===k[0].offsetHeight,f&&(k[0].style.display="",k[1].style.display="none",f=0===k[0].offsetHeight)),m.removeChild(i)}}}();var Ra,Sa,Ta=/^(top|right|bottom|left)$/;a.getComputedStyle?(Ra=function(b){var c=b.ownerDocument.defaultView;return c&&c.opener||(c=a),c.getComputedStyle(b)},Sa=function(a,b,c){var d,e,f,g,h=a.style;return c=c||Ra(a),g=c?c.getPropertyValue(b)||c[b]:void 0,""!==g&&void 0!==g||n.contains(a.ownerDocument,a)||(g=n.style(a,b)),c&&!l.pixelMarginRight()&&Oa.test(g)&&Na.test(b)&&(d=h.width,e=h.minWidth,f=h.maxWidth,h.minWidth=h.maxWidth=h.width=g,g=c.width,h.width=d,h.minWidth=e,h.maxWidth=f),void 0===g?g:g+""}):Qa.currentStyle&&(Ra=function(a){return a.currentStyle},Sa=function(a,b,c){var d,e,f,g,h=a.style;return c=c||Ra(a),g=c?c[b]:void 0,null==g&&h&&h[b]&&(g=h[b]),Oa.test(g)&&!Ta.test(b)&&(d=h.left,e=a.runtimeStyle,f=e&&e.left,f&&(e.left=a.currentStyle.left),h.left="fontSize"===b?"1em":g,g=h.pixelLeft+"px",h.left=d,f&&(e.left=f)),void 0===g?g:g+""||"auto"});function Ua(a,b){return{get:function(){return a()?void delete this.get:(this.get=b).apply(this,arguments)}}}var Va=/alpha\([^)]*\)/i,Wa=/opacity\s*=\s*([^)]*)/i,Xa=/^(none|table(?!-c[ea]).+)/,Ya=new RegExp("^("+T+")(.*)$","i"),Za={position:"absolute",visibility:"hidden",display:"block"},$a={letterSpacing:"0",fontWeight:"400"},_a=["Webkit","O","Moz","ms"],ab=d.createElement("div").style;function bb(a){if(a in ab)return a;var b=a.charAt(0).toUpperCase()+a.slice(1),c=_a.length;while(c--)if(a=_a[c]+b,a in ab)return a}function cb(a,b){for(var c,d,e,f=[],g=0,h=a.length;h>g;g++)d=a[g],d.style&&(f[g]=n._data(d,"olddisplay"),c=d.style.display,b?(f[g]||"none"!==c||(d.style.display=""),""===d.style.display&&W(d)&&(f[g]=n._data(d,"olddisplay",Ma(d.nodeName)))):(e=W(d),(c&&"none"!==c||!e)&&n._data(d,"olddisplay",e?c:n.css(d,"display"))));for(g=0;h>g;g++)d=a[g],d.style&&(b&&"none"!==d.style.display&&""!==d.style.display||(d.style.display=b?f[g]||"":"none"));return a}function db(a,b,c){var d=Ya.exec(b);return d?Math.max(0,d[1]-(c||0))+(d[2]||"px"):b}function eb(a,b,c,d,e){for(var f=c===(d?"border":"content")?4:"width"===b?1:0,g=0;4>f;f+=2)"margin"===c&&(g+=n.css(a,c+V[f],!0,e)),d?("content"===c&&(g-=n.css(a,"padding"+V[f],!0,e)),"margin"!==c&&(g-=n.css(a,"border"+V[f]+"Width",!0,e))):(g+=n.css(a,"padding"+V[f],!0,e),"padding"!==c&&(g+=n.css(a,"border"+V[f]+"Width",!0,e)));return g}function fb(a,b,c){var d=!0,e="width"===b?a.offsetWidth:a.offsetHeight,f=Ra(a),g=l.boxSizing&&"border-box"===n.css(a,"boxSizing",!1,f);if(0>=e||null==e){if(e=Sa(a,b,f),(0>e||null==e)&&(e=a.style[b]),Oa.test(e))return e;d=g&&(l.boxSizingReliable()||e===a.style[b]),e=parseFloat(e)||0}return e+eb(a,b,c||(g?"border":"content"),d,f)+"px"}n.extend({cssHooks:{opacity:{get:function(a,b){if(b){var c=Sa(a,"opacity");return""===c?"1":c}}}},cssNumber:{animationIterationCount:!0,columnCount:!0,fillOpacity:!0,flexGrow:!0,flexShrink:!0,fontWeight:!0,lineHeight:!0,opacity:!0,order:!0,orphans:!0,widows:!0,zIndex:!0,zoom:!0},cssProps:{"float":l.cssFloat?"cssFloat":"styleFloat"},style:function(a,b,c,d){if(a&&3!==a.nodeType&&8!==a.nodeType&&a.style){var e,f,g,h=n.camelCase(b),i=a.style;if(b=n.cssProps[h]||(n.cssProps[h]=bb(h)||h),g=n.cssHooks[b]||n.cssHooks[h],void 0===c)return g&&"get"in g&&void 0!==(e=g.get(a,!1,d))?e:i[b];if(f=typeof c,"string"===f&&(e=U.exec(c))&&e[1]&&(c=X(a,b,e),f="number"),null!=c&&c===c&&("number"===f&&(c+=e&&e[3]||(n.cssNumber[h]?"":"px")),l.clearCloneStyle||""!==c||0!==b.indexOf("background")||(i[b]="inherit"),!(g&&"set"in g&&void 0===(c=g.set(a,c,d)))))try{i[b]=c}catch(j){}}},css:function(a,b,c,d){var e,f,g,h=n.camelCase(b);return b=n.cssProps[h]||(n.cssProps[h]=bb(h)||h),g=n.cssHooks[b]||n.cssHooks[h],g&&"get"in g&&(f=g.get(a,!0,c)),void 0===f&&(f=Sa(a,b,d)),"normal"===f&&b in $a&&(f=$a[b]),""===c||c?(e=parseFloat(f),c===!0||isFinite(e)?e||0:f):f}}),n.each(["height","width"],function(a,b){n.cssHooks[b]={get:function(a,c,d){return c?Xa.test(n.css(a,"display"))&&0===a.offsetWidth?Pa(a,Za,function(){return fb(a,b,d)}):fb(a,b,d):void 0},set:function(a,c,d){var e=d&&Ra(a);return db(a,c,d?eb(a,b,d,l.boxSizing&&"border-box"===n.css(a,"boxSizing",!1,e),e):0)}}}),l.opacity||(n.cssHooks.opacity={get:function(a,b){return Wa.test((b&&a.currentStyle?a.currentStyle.filter:a.style.filter)||"")?.01*parseFloat(RegExp.$1)+"":b?"1":""},set:function(a,b){var c=a.style,d=a.currentStyle,e=n.isNumeric(b)?"alpha(opacity="+100*b+")":"",f=d&&d.filter||c.filter||"";c.zoom=1,(b>=1||""===b)&&""===n.trim(f.replace(Va,""))&&c.removeAttribute&&(c.removeAttribute("filter"),""===b||d&&!d.filter)||(c.filter=Va.test(f)?f.replace(Va,e):f+" "+e)}}),n.cssHooks.marginRight=Ua(l.reliableMarginRight,function(a,b){return b?Pa(a,{display:"inline-block"},Sa,[a,"marginRight"]):void 0}),n.cssHooks.marginLeft=Ua(l.reliableMarginLeft,function(a,b){return b?(parseFloat(Sa(a,"marginLeft"))||(n.contains(a.ownerDocument,a)?a.getBoundingClientRect().left-Pa(a,{
marginLeft:0},function(){return a.getBoundingClientRect().left}):0))+"px":void 0}),n.each({margin:"",padding:"",border:"Width"},function(a,b){n.cssHooks[a+b]={expand:function(c){for(var d=0,e={},f="string"==typeof c?c.split(" "):[c];4>d;d++)e[a+V[d]+b]=f[d]||f[d-2]||f[0];return e}},Na.test(a)||(n.cssHooks[a+b].set=db)}),n.fn.extend({css:function(a,b){return Y(this,function(a,b,c){var d,e,f={},g=0;if(n.isArray(b)){for(d=Ra(a),e=b.length;e>g;g++)f[b[g]]=n.css(a,b[g],!1,d);return f}return void 0!==c?n.style(a,b,c):n.css(a,b)},a,b,arguments.length>1)},show:function(){return cb(this,!0)},hide:function(){return cb(this)},toggle:function(a){return"boolean"==typeof a?a?this.show():this.hide():this.each(function(){W(this)?n(this).show():n(this).hide()})}});function gb(a,b,c,d,e){return new gb.prototype.init(a,b,c,d,e)}n.Tween=gb,gb.prototype={constructor:gb,init:function(a,b,c,d,e,f){this.elem=a,this.prop=c,this.easing=e||n.easing._default,this.options=b,this.start=this.now=this.cur(),this.end=d,this.unit=f||(n.cssNumber[c]?"":"px")},cur:function(){var a=gb.propHooks[this.prop];return a&&a.get?a.get(this):gb.propHooks._default.get(this)},run:function(a){var b,c=gb.propHooks[this.prop];return this.options.duration?this.pos=b=n.easing[this.easing](a,this.options.duration*a,0,1,this.options.duration):this.pos=b=a,this.now=(this.end-this.start)*b+this.start,this.options.step&&this.options.step.call(this.elem,this.now,this),c&&c.set?c.set(this):gb.propHooks._default.set(this),this}},gb.prototype.init.prototype=gb.prototype,gb.propHooks={_default:{get:function(a){var b;return 1!==a.elem.nodeType||null!=a.elem[a.prop]&&null==a.elem.style[a.prop]?a.elem[a.prop]:(b=n.css(a.elem,a.prop,""),b&&"auto"!==b?b:0)},set:function(a){n.fx.step[a.prop]?n.fx.step[a.prop](a):1!==a.elem.nodeType||null==a.elem.style[n.cssProps[a.prop]]&&!n.cssHooks[a.prop]?a.elem[a.prop]=a.now:n.style(a.elem,a.prop,a.now+a.unit)}}},gb.propHooks.scrollTop=gb.propHooks.scrollLeft={set:function(a){a.elem.nodeType&&a.elem.parentNode&&(a.elem[a.prop]=a.now)}},n.easing={linear:function(a){return a},swing:function(a){return.5-Math.cos(a*Math.PI)/2},_default:"swing"},n.fx=gb.prototype.init,n.fx.step={};var hb,ib,jb=/^(?:toggle|show|hide)$/,kb=/queueHooks$/;function lb(){return a.setTimeout(function(){hb=void 0}),hb=n.now()}function mb(a,b){var c,d={height:a},e=0;for(b=b?1:0;4>e;e+=2-b)c=V[e],d["margin"+c]=d["padding"+c]=a;return b&&(d.opacity=d.width=a),d}function nb(a,b,c){for(var d,e=(qb.tweeners[b]||[]).concat(qb.tweeners["*"]),f=0,g=e.length;g>f;f++)if(d=e[f].call(c,b,a))return d}function ob(a,b,c){var d,e,f,g,h,i,j,k,m=this,o={},p=a.style,q=a.nodeType&&W(a),r=n._data(a,"fxshow");c.queue||(h=n._queueHooks(a,"fx"),null==h.unqueued&&(h.unqueued=0,i=h.empty.fire,h.empty.fire=function(){h.unqueued||i()}),h.unqueued++,m.always(function(){m.always(function(){h.unqueued--,n.queue(a,"fx").length||h.empty.fire()})})),1===a.nodeType&&("height"in b||"width"in b)&&(c.overflow=[p.overflow,p.overflowX,p.overflowY],j=n.css(a,"display"),k="none"===j?n._data(a,"olddisplay")||Ma(a.nodeName):j,"inline"===k&&"none"===n.css(a,"float")&&(l.inlineBlockNeedsLayout&&"inline"!==Ma(a.nodeName)?p.zoom=1:p.display="inline-block")),c.overflow&&(p.overflow="hidden",l.shrinkWrapBlocks()||m.always(function(){p.overflow=c.overflow[0],p.overflowX=c.overflow[1],p.overflowY=c.overflow[2]}));for(d in b)if(e=b[d],jb.exec(e)){if(delete b[d],f=f||"toggle"===e,e===(q?"hide":"show")){if("show"!==e||!r||void 0===r[d])continue;q=!0}o[d]=r&&r[d]||n.style(a,d)}else j=void 0;if(n.isEmptyObject(o))"inline"===("none"===j?Ma(a.nodeName):j)&&(p.display=j);else{r?"hidden"in r&&(q=r.hidden):r=n._data(a,"fxshow",{}),f&&(r.hidden=!q),q?n(a).show():m.done(function(){n(a).hide()}),m.done(function(){var b;n._removeData(a,"fxshow");for(b in o)n.style(a,b,o[b])});for(d in o)g=nb(q?r[d]:0,d,m),d in r||(r[d]=g.start,q&&(g.end=g.start,g.start="width"===d||"height"===d?1:0))}}function pb(a,b){var c,d,e,f,g;for(c in a)if(d=n.camelCase(c),e=b[d],f=a[c],n.isArray(f)&&(e=f[1],f=a[c]=f[0]),c!==d&&(a[d]=f,delete a[c]),g=n.cssHooks[d],g&&"expand"in g){f=g.expand(f),delete a[d];for(c in f)c in a||(a[c]=f[c],b[c]=e)}else b[d]=e}function qb(a,b,c){var d,e,f=0,g=qb.prefilters.length,h=n.Deferred().always(function(){delete i.elem}),i=function(){if(e)return!1;for(var b=hb||lb(),c=Math.max(0,j.startTime+j.duration-b),d=c/j.duration||0,f=1-d,g=0,i=j.tweens.length;i>g;g++)j.tweens[g].run(f);return h.notifyWith(a,[j,f,c]),1>f&&i?c:(h.resolveWith(a,[j]),!1)},j=h.promise({elem:a,props:n.extend({},b),opts:n.extend(!0,{specialEasing:{},easing:n.easing._default},c),originalProperties:b,originalOptions:c,startTime:hb||lb(),duration:c.duration,tweens:[],createTween:function(b,c){var d=n.Tween(a,j.opts,b,c,j.opts.specialEasing[b]||j.opts.easing);return j.tweens.push(d),d},stop:function(b){var c=0,d=b?j.tweens.length:0;if(e)return this;for(e=!0;d>c;c++)j.tweens[c].run(1);return b?(h.notifyWith(a,[j,1,0]),h.resolveWith(a,[j,b])):h.rejectWith(a,[j,b]),this}}),k=j.props;for(pb(k,j.opts.specialEasing);g>f;f++)if(d=qb.prefilters[f].call(j,a,k,j.opts))return n.isFunction(d.stop)&&(n._queueHooks(j.elem,j.opts.queue).stop=n.proxy(d.stop,d)),d;return n.map(k,nb,j),n.isFunction(j.opts.start)&&j.opts.start.call(a,j),n.fx.timer(n.extend(i,{elem:a,anim:j,queue:j.opts.queue})),j.progress(j.opts.progress).done(j.opts.done,j.opts.complete).fail(j.opts.fail).always(j.opts.always)}n.Animation=n.extend(qb,{tweeners:{"*":[function(a,b){var c=this.createTween(a,b);return X(c.elem,a,U.exec(b),c),c}]},tweener:function(a,b){n.isFunction(a)?(b=a,a=["*"]):a=a.match(G);for(var c,d=0,e=a.length;e>d;d++)c=a[d],qb.tweeners[c]=qb.tweeners[c]||[],qb.tweeners[c].unshift(b)},prefilters:[ob],prefilter:function(a,b){b?qb.prefilters.unshift(a):qb.prefilters.push(a)}}),n.speed=function(a,b,c){var d=a&&"object"==typeof a?n.extend({},a):{complete:c||!c&&b||n.isFunction(a)&&a,duration:a,easing:c&&b||b&&!n.isFunction(b)&&b};return d.duration=n.fx.off?0:"number"==typeof d.duration?d.duration:d.duration in n.fx.speeds?n.fx.speeds[d.duration]:n.fx.speeds._default,null!=d.queue&&d.queue!==!0||(d.queue="fx"),d.old=d.complete,d.complete=function(){n.isFunction(d.old)&&d.old.call(this),d.queue&&n.dequeue(this,d.queue)},d},n.fn.extend({fadeTo:function(a,b,c,d){return this.filter(W).css("opacity",0).show().end().animate({opacity:b},a,c,d)},animate:function(a,b,c,d){var e=n.isEmptyObject(a),f=n.speed(b,c,d),g=function(){var b=qb(this,n.extend({},a),f);(e||n._data(this,"finish"))&&b.stop(!0)};return g.finish=g,e||f.queue===!1?this.each(g):this.queue(f.queue,g)},stop:function(a,b,c){var d=function(a){var b=a.stop;delete a.stop,b(c)};return"string"!=typeof a&&(c=b,b=a,a=void 0),b&&a!==!1&&this.queue(a||"fx",[]),this.each(function(){var b=!0,e=null!=a&&a+"queueHooks",f=n.timers,g=n._data(this);if(e)g[e]&&g[e].stop&&d(g[e]);else for(e in g)g[e]&&g[e].stop&&kb.test(e)&&d(g[e]);for(e=f.length;e--;)f[e].elem!==this||null!=a&&f[e].queue!==a||(f[e].anim.stop(c),b=!1,f.splice(e,1));!b&&c||n.dequeue(this,a)})},finish:function(a){return a!==!1&&(a=a||"fx"),this.each(function(){var b,c=n._data(this),d=c[a+"queue"],e=c[a+"queueHooks"],f=n.timers,g=d?d.length:0;for(c.finish=!0,n.queue(this,a,[]),e&&e.stop&&e.stop.call(this,!0),b=f.length;b--;)f[b].elem===this&&f[b].queue===a&&(f[b].anim.stop(!0),f.splice(b,1));for(b=0;g>b;b++)d[b]&&d[b].finish&&d[b].finish.call(this);delete c.finish})}}),n.each(["toggle","show","hide"],function(a,b){var c=n.fn[b];n.fn[b]=function(a,d,e){return null==a||"boolean"==typeof a?c.apply(this,arguments):this.animate(mb(b,!0),a,d,e)}}),n.each({slideDown:mb("show"),slideUp:mb("hide"),slideToggle:mb("toggle"),fadeIn:{opacity:"show"},fadeOut:{opacity:"hide"},fadeToggle:{opacity:"toggle"}},function(a,b){n.fn[a]=function(a,c,d){return this.animate(b,a,c,d)}}),n.timers=[],n.fx.tick=function(){var a,b=n.timers,c=0;for(hb=n.now();c<b.length;c++)a=b[c],a()||b[c]!==a||b.splice(c--,1);b.length||n.fx.stop(),hb=void 0},n.fx.timer=function(a){n.timers.push(a),a()?n.fx.start():n.timers.pop()},n.fx.interval=13,n.fx.start=function(){ib||(ib=a.setInterval(n.fx.tick,n.fx.interval))},n.fx.stop=function(){a.clearInterval(ib),ib=null},n.fx.speeds={slow:600,fast:200,_default:400},n.fn.delay=function(b,c){return b=n.fx?n.fx.speeds[b]||b:b,c=c||"fx",this.queue(c,function(c,d){var e=a.setTimeout(c,b);d.stop=function(){a.clearTimeout(e)}})},function(){var a,b=d.createElement("input"),c=d.createElement("div"),e=d.createElement("select"),f=e.appendChild(d.createElement("option"));c=d.createElement("div"),c.setAttribute("className","t"),c.innerHTML="  <link/><table></table><a href='/a'>a</a><input type='checkbox'/>",a=c.getElementsByTagName("a")[0],b.setAttribute("type","checkbox"),c.appendChild(b),a=c.getElementsByTagName("a")[0],a.style.cssText="top:1px",l.getSetAttribute="t"!==c.className,l.style=/top/.test(a.getAttribute("style")),l.hrefNormalized="/a"===a.getAttribute("href"),l.checkOn=!!b.value,l.optSelected=f.selected,l.enctype=!!d.createElement("form").enctype,e.disabled=!0,l.optDisabled=!f.disabled,b=d.createElement("input"),b.setAttribute("value",""),l.input=""===b.getAttribute("value"),b.value="t",b.setAttribute("type","radio"),l.radioValue="t"===b.value}();var rb=/\r/g,sb=/[\x20\t\r\n\f]+/g;n.fn.extend({val:function(a){var b,c,d,e=this[0];{if(arguments.length)return d=n.isFunction(a),this.each(function(c){var e;1===this.nodeType&&(e=d?a.call(this,c,n(this).val()):a,null==e?e="":"number"==typeof e?e+="":n.isArray(e)&&(e=n.map(e,function(a){return null==a?"":a+""})),b=n.valHooks[this.type]||n.valHooks[this.nodeName.toLowerCase()],b&&"set"in b&&void 0!==b.set(this,e,"value")||(this.value=e))});if(e)return b=n.valHooks[e.type]||n.valHooks[e.nodeName.toLowerCase()],b&&"get"in b&&void 0!==(c=b.get(e,"value"))?c:(c=e.value,"string"==typeof c?c.replace(rb,""):null==c?"":c)}}}),n.extend({valHooks:{option:{get:function(a){var b=n.find.attr(a,"value");return null!=b?b:n.trim(n.text(a)).replace(sb," ")}},select:{get:function(a){for(var b,c,d=a.options,e=a.selectedIndex,f="select-one"===a.type||0>e,g=f?null:[],h=f?e+1:d.length,i=0>e?h:f?e:0;h>i;i++)if(c=d[i],(c.selected||i===e)&&(l.optDisabled?!c.disabled:null===c.getAttribute("disabled"))&&(!c.parentNode.disabled||!n.nodeName(c.parentNode,"optgroup"))){if(b=n(c).val(),f)return b;g.push(b)}return g},set:function(a,b){var c,d,e=a.options,f=n.makeArray(b),g=e.length;while(g--)if(d=e[g],n.inArray(n.valHooks.option.get(d),f)>-1)try{d.selected=c=!0}catch(h){d.scrollHeight}else d.selected=!1;return c||(a.selectedIndex=-1),e}}}}),n.each(["radio","checkbox"],function(){n.valHooks[this]={set:function(a,b){return n.isArray(b)?a.checked=n.inArray(n(a).val(),b)>-1:void 0}},l.checkOn||(n.valHooks[this].get=function(a){return null===a.getAttribute("value")?"on":a.value})});var tb,ub,vb=n.expr.attrHandle,wb=/^(?:checked|selected)$/i,xb=l.getSetAttribute,yb=l.input;n.fn.extend({attr:function(a,b){return Y(this,n.attr,a,b,arguments.length>1)},removeAttr:function(a){return this.each(function(){n.removeAttr(this,a)})}}),n.extend({attr:function(a,b,c){var d,e,f=a.nodeType;if(3!==f&&8!==f&&2!==f)return"undefined"==typeof a.getAttribute?n.prop(a,b,c):(1===f&&n.isXMLDoc(a)||(b=b.toLowerCase(),e=n.attrHooks[b]||(n.expr.match.bool.test(b)?ub:tb)),void 0!==c?null===c?void n.removeAttr(a,b):e&&"set"in e&&void 0!==(d=e.set(a,c,b))?d:(a.setAttribute(b,c+""),c):e&&"get"in e&&null!==(d=e.get(a,b))?d:(d=n.find.attr(a,b),null==d?void 0:d))},attrHooks:{type:{set:function(a,b){if(!l.radioValue&&"radio"===b&&n.nodeName(a,"input")){var c=a.value;return a.setAttribute("type",b),c&&(a.value=c),b}}}},removeAttr:function(a,b){var c,d,e=0,f=b&&b.match(G);if(f&&1===a.nodeType)while(c=f[e++])d=n.propFix[c]||c,n.expr.match.bool.test(c)?yb&&xb||!wb.test(c)?a[d]=!1:a[n.camelCase("default-"+c)]=a[d]=!1:n.attr(a,c,""),a.removeAttribute(xb?c:d)}}),ub={set:function(a,b,c){return b===!1?n.removeAttr(a,c):yb&&xb||!wb.test(c)?a.setAttribute(!xb&&n.propFix[c]||c,c):a[n.camelCase("default-"+c)]=a[c]=!0,c}},n.each(n.expr.match.bool.source.match(/\w+/g),function(a,b){var c=vb[b]||n.find.attr;yb&&xb||!wb.test(b)?vb[b]=function(a,b,d){var e,f;return d||(f=vb[b],vb[b]=e,e=null!=c(a,b,d)?b.toLowerCase():null,vb[b]=f),e}:vb[b]=function(a,b,c){return c?void 0:a[n.camelCase("default-"+b)]?b.toLowerCase():null}}),yb&&xb||(n.attrHooks.value={set:function(a,b,c){return n.nodeName(a,"input")?void(a.defaultValue=b):tb&&tb.set(a,b,c)}}),xb||(tb={set:function(a,b,c){var d=a.getAttributeNode(c);return d||a.setAttributeNode(d=a.ownerDocument.createAttribute(c)),d.value=b+="","value"===c||b===a.getAttribute(c)?b:void 0}},vb.id=vb.name=vb.coords=function(a,b,c){var d;return c?void 0:(d=a.getAttributeNode(b))&&""!==d.value?d.value:null},n.valHooks.button={get:function(a,b){var c=a.getAttributeNode(b);return c&&c.specified?c.value:void 0},set:tb.set},n.attrHooks.contenteditable={set:function(a,b,c){tb.set(a,""===b?!1:b,c)}},n.each(["width","height"],function(a,b){n.attrHooks[b]={set:function(a,c){return""===c?(a.setAttribute(b,"auto"),c):void 0}}})),l.style||(n.attrHooks.style={get:function(a){return a.style.cssText||void 0},set:function(a,b){return a.style.cssText=b+""}});var zb=/^(?:input|select|textarea|button|object)$/i,Ab=/^(?:a|area)$/i;n.fn.extend({prop:function(a,b){return Y(this,n.prop,a,b,arguments.length>1)},removeProp:function(a){return a=n.propFix[a]||a,this.each(function(){try{this[a]=void 0,delete this[a]}catch(b){}})}}),n.extend({prop:function(a,b,c){var d,e,f=a.nodeType;if(3!==f&&8!==f&&2!==f)return 1===f&&n.isXMLDoc(a)||(b=n.propFix[b]||b,e=n.propHooks[b]),void 0!==c?e&&"set"in e&&void 0!==(d=e.set(a,c,b))?d:a[b]=c:e&&"get"in e&&null!==(d=e.get(a,b))?d:a[b]},propHooks:{tabIndex:{get:function(a){var b=n.find.attr(a,"tabindex");return b?parseInt(b,10):zb.test(a.nodeName)||Ab.test(a.nodeName)&&a.href?0:-1}}},propFix:{"for":"htmlFor","class":"className"}}),l.hrefNormalized||n.each(["href","src"],function(a,b){n.propHooks[b]={get:function(a){return a.getAttribute(b,4)}}}),l.optSelected||(n.propHooks.selected={get:function(a){var b=a.parentNode;return b&&(b.selectedIndex,b.parentNode&&b.parentNode.selectedIndex),null},set:function(a){var b=a.parentNode;b&&(b.selectedIndex,b.parentNode&&b.parentNode.selectedIndex)}}),n.each(["tabIndex","readOnly","maxLength","cellSpacing","cellPadding","rowSpan","colSpan","useMap","frameBorder","contentEditable"],function(){n.propFix[this.toLowerCase()]=this}),l.enctype||(n.propFix.enctype="encoding");var Bb=/[\t\r\n\f]/g;function Cb(a){return n.attr(a,"class")||""}n.fn.extend({addClass:function(a){var b,c,d,e,f,g,h,i=0;if(n.isFunction(a))return this.each(function(b){n(this).addClass(a.call(this,b,Cb(this)))});if("string"==typeof a&&a){b=a.match(G)||[];while(c=this[i++])if(e=Cb(c),d=1===c.nodeType&&(" "+e+" ").replace(Bb," ")){g=0;while(f=b[g++])d.indexOf(" "+f+" ")<0&&(d+=f+" ");h=n.trim(d),e!==h&&n.attr(c,"class",h)}}return this},removeClass:function(a){var b,c,d,e,f,g,h,i=0;if(n.isFunction(a))return this.each(function(b){n(this).removeClass(a.call(this,b,Cb(this)))});if(!arguments.length)return this.attr("class","");if("string"==typeof a&&a){b=a.match(G)||[];while(c=this[i++])if(e=Cb(c),d=1===c.nodeType&&(" "+e+" ").replace(Bb," ")){g=0;while(f=b[g++])while(d.indexOf(" "+f+" ")>-1)d=d.replace(" "+f+" "," ");h=n.trim(d),e!==h&&n.attr(c,"class",h)}}return this},toggleClass:function(a,b){var c=typeof a;return"boolean"==typeof b&&"string"===c?b?this.addClass(a):this.removeClass(a):n.isFunction(a)?this.each(function(c){n(this).toggleClass(a.call(this,c,Cb(this),b),b)}):this.each(function(){var b,d,e,f;if("string"===c){d=0,e=n(this),f=a.match(G)||[];while(b=f[d++])e.hasClass(b)?e.removeClass(b):e.addClass(b)}else void 0!==a&&"boolean"!==c||(b=Cb(this),b&&n._data(this,"__className__",b),n.attr(this,"class",b||a===!1?"":n._data(this,"__className__")||""))})},hasClass:function(a){var b,c,d=0;b=" "+a+" ";while(c=this[d++])if(1===c.nodeType&&(" "+Cb(c)+" ").replace(Bb," ").indexOf(b)>-1)return!0;return!1}}),n.each("blur focus focusin focusout load resize scroll unload click dblclick mousedown mouseup mousemove mouseover mouseout mouseenter mouseleave change select submit keydown keypress keyup error contextmenu".split(" "),function(a,b){n.fn[b]=function(a,c){return arguments.length>0?this.on(b,null,a,c):this.trigger(b)}}),n.fn.extend({hover:function(a,b){return this.mouseenter(a).mouseleave(b||a)}});var Db=a.location,Eb=n.now(),Fb=/\?/,Gb=/(,)|(\[|{)|(}|])|"(?:[^"\\\r\n]|\\["\\\/bfnrt]|\\u[\da-fA-F]{4})*"\s*:?|true|false|null|-?(?!0\d)\d+(?:\.\d+|)(?:[eE][+-]?\d+|)/g;n.parseJSON=function(b){if(a.JSON&&a.JSON.parse)return a.JSON.parse(b+"");var c,d=null,e=n.trim(b+"");return e&&!n.trim(e.replace(Gb,function(a,b,e,f){return c&&b&&(d=0),0===d?a:(c=e||b,d+=!f-!e,"")}))?Function("return "+e)():n.error("Invalid JSON: "+b)},n.parseXML=function(b){var c,d;if(!b||"string"!=typeof b)return null;try{a.DOMParser?(d=new a.DOMParser,c=d.parseFromString(b,"text/xml")):(c=new a.ActiveXObject("Microsoft.XMLDOM"),c.async="false",c.loadXML(b))}catch(e){c=void 0}return c&&c.documentElement&&!c.getElementsByTagName("parsererror").length||n.error("Invalid XML: "+b),c};var Hb=/#.*$/,Ib=/([?&])_=[^&]*/,Jb=/^(.*?):[ \t]*([^\r\n]*)\r?$/gm,Kb=/^(?:about|app|app-storage|.+-extension|file|res|widget):$/,Lb=/^(?:GET|HEAD)$/,Mb=/^\/\//,Nb=/^([\w.+-]+:)(?:\/\/(?:[^\/?#]*@|)([^\/?#:]*)(?::(\d+)|)|)/,Ob={},Pb={},Qb="*/".concat("*"),Rb=Db.href,Sb=Nb.exec(Rb.toLowerCase())||[];function Tb(a){return function(b,c){"string"!=typeof b&&(c=b,b="*");var d,e=0,f=b.toLowerCase().match(G)||[];if(n.isFunction(c))while(d=f[e++])"+"===d.charAt(0)?(d=d.slice(1)||"*",(a[d]=a[d]||[]).unshift(c)):(a[d]=a[d]||[]).push(c)}}function Ub(a,b,c,d){var e={},f=a===Pb;function g(h){var i;return e[h]=!0,n.each(a[h]||[],function(a,h){var j=h(b,c,d);return"string"!=typeof j||f||e[j]?f?!(i=j):void 0:(b.dataTypes.unshift(j),g(j),!1)}),i}return g(b.dataTypes[0])||!e["*"]&&g("*")}function Vb(a,b){var c,d,e=n.ajaxSettings.flatOptions||{};for(d in b)void 0!==b[d]&&((e[d]?a:c||(c={}))[d]=b[d]);return c&&n.extend(!0,a,c),a}function Wb(a,b,c){var d,e,f,g,h=a.contents,i=a.dataTypes;while("*"===i[0])i.shift(),void 0===e&&(e=a.mimeType||b.getResponseHeader("Content-Type"));if(e)for(g in h)if(h[g]&&h[g].test(e)){i.unshift(g);break}if(i[0]in c)f=i[0];else{for(g in c){if(!i[0]||a.converters[g+" "+i[0]]){f=g;break}d||(d=g)}f=f||d}return f?(f!==i[0]&&i.unshift(f),c[f]):void 0}function Xb(a,b,c,d){var e,f,g,h,i,j={},k=a.dataTypes.slice();if(k[1])for(g in a.converters)j[g.toLowerCase()]=a.converters[g];f=k.shift();while(f)if(a.responseFields[f]&&(c[a.responseFields[f]]=b),!i&&d&&a.dataFilter&&(b=a.dataFilter(b,a.dataType)),i=f,f=k.shift())if("*"===f)f=i;else if("*"!==i&&i!==f){if(g=j[i+" "+f]||j["* "+f],!g)for(e in j)if(h=e.split(" "),h[1]===f&&(g=j[i+" "+h[0]]||j["* "+h[0]])){g===!0?g=j[e]:j[e]!==!0&&(f=h[0],k.unshift(h[1]));break}if(g!==!0)if(g&&a["throws"])b=g(b);else try{b=g(b)}catch(l){return{state:"parsererror",error:g?l:"No conversion from "+i+" to "+f}}}return{state:"success",data:b}}n.extend({active:0,lastModified:{},etag:{},ajaxSettings:{url:Rb,type:"GET",isLocal:Kb.test(Sb[1]),global:!0,processData:!0,async:!0,contentType:"application/x-www-form-urlencoded; charset=UTF-8",accepts:{"*":Qb,text:"text/plain",html:"text/html",xml:"application/xml, text/xml",json:"application/json, text/javascript"},contents:{xml:/\bxml\b/,html:/\bhtml/,json:/\bjson\b/},responseFields:{xml:"responseXML",text:"responseText",json:"responseJSON"},converters:{"* text":String,"text html":!0,"text json":n.parseJSON,"text xml":n.parseXML},flatOptions:{url:!0,context:!0}},ajaxSetup:function(a,b){return b?Vb(Vb(a,n.ajaxSettings),b):Vb(n.ajaxSettings,a)},ajaxPrefilter:Tb(Ob),ajaxTransport:Tb(Pb),ajax:function(b,c){"object"==typeof b&&(c=b,b=void 0),c=c||{};var d,e,f,g,h,i,j,k,l=n.ajaxSetup({},c),m=l.context||l,o=l.context&&(m.nodeType||m.jquery)?n(m):n.event,p=n.Deferred(),q=n.Callbacks("once memory"),r=l.statusCode||{},s={},t={},u=0,v="canceled",w={readyState:0,getResponseHeader:function(a){var b;if(2===u){if(!k){k={};while(b=Jb.exec(g))k[b[1].toLowerCase()]=b[2]}b=k[a.toLowerCase()]}return null==b?null:b},getAllResponseHeaders:function(){return 2===u?g:null},setRequestHeader:function(a,b){var c=a.toLowerCase();return u||(a=t[c]=t[c]||a,s[a]=b),this},overrideMimeType:function(a){return u||(l.mimeType=a),this},statusCode:function(a){var b;if(a)if(2>u)for(b in a)r[b]=[r[b],a[b]];else w.always(a[w.status]);return this},abort:function(a){var b=a||v;return j&&j.abort(b),y(0,b),this}};if(p.promise(w).complete=q.add,w.success=w.done,w.error=w.fail,l.url=((b||l.url||Rb)+"").replace(Hb,"").replace(Mb,Sb[1]+"//"),l.type=c.method||c.type||l.method||l.type,l.dataTypes=n.trim(l.dataType||"*").toLowerCase().match(G)||[""],null==l.crossDomain&&(d=Nb.exec(l.url.toLowerCase()),l.crossDomain=!(!d||d[1]===Sb[1]&&d[2]===Sb[2]&&(d[3]||("http:"===d[1]?"80":"443"))===(Sb[3]||("http:"===Sb[1]?"80":"443")))),l.data&&l.processData&&"string"!=typeof l.data&&(l.data=n.param(l.data,l.traditional)),Ub(Ob,l,c,w),2===u)return w;i=n.event&&l.global,i&&0===n.active++&&n.event.trigger("ajaxStart"),l.type=l.type.toUpperCase(),l.hasContent=!Lb.test(l.type),f=l.url,l.hasContent||(l.data&&(f=l.url+=(Fb.test(f)?"&":"?")+l.data,delete l.data),l.cache===!1&&(l.url=Ib.test(f)?f.replace(Ib,"$1_="+Eb++):f+(Fb.test(f)?"&":"?")+"_="+Eb++)),l.ifModified&&(n.lastModified[f]&&w.setRequestHeader("If-Modified-Since",n.lastModified[f]),n.etag[f]&&w.setRequestHeader("If-None-Match",n.etag[f])),(l.data&&l.hasContent&&l.contentType!==!1||c.contentType)&&w.setRequestHeader("Content-Type",l.contentType),w.setRequestHeader("Accept",l.dataTypes[0]&&l.accepts[l.dataTypes[0]]?l.accepts[l.dataTypes[0]]+("*"!==l.dataTypes[0]?", "+Qb+"; q=0.01":""):l.accepts["*"]);for(e in l.headers)w.setRequestHeader(e,l.headers[e]);if(l.beforeSend&&(l.beforeSend.call(m,w,l)===!1||2===u))return w.abort();v="abort";for(e in{success:1,error:1,complete:1})w[e](l[e]);if(j=Ub(Pb,l,c,w)){if(w.readyState=1,i&&o.trigger("ajaxSend",[w,l]),2===u)return w;l.async&&l.timeout>0&&(h=a.setTimeout(function(){w.abort("timeout")},l.timeout));try{u=1,j.send(s,y)}catch(x){if(!(2>u))throw x;y(-1,x)}}else y(-1,"No Transport");function y(b,c,d,e){var k,s,t,v,x,y=c;2!==u&&(u=2,h&&a.clearTimeout(h),j=void 0,g=e||"",w.readyState=b>0?4:0,k=b>=200&&300>b||304===b,d&&(v=Wb(l,w,d)),v=Xb(l,v,w,k),k?(l.ifModified&&(x=w.getResponseHeader("Last-Modified"),x&&(n.lastModified[f]=x),x=w.getResponseHeader("etag"),x&&(n.etag[f]=x)),204===b||"HEAD"===l.type?y="nocontent":304===b?y="notmodified":(y=v.state,s=v.data,t=v.error,k=!t)):(t=y,!b&&y||(y="error",0>b&&(b=0))),w.status=b,w.statusText=(c||y)+"",k?p.resolveWith(m,[s,y,w]):p.rejectWith(m,[w,y,t]),w.statusCode(r),r=void 0,i&&o.trigger(k?"ajaxSuccess":"ajaxError",[w,l,k?s:t]),q.fireWith(m,[w,y]),i&&(o.trigger("ajaxComplete",[w,l]),--n.active||n.event.trigger("ajaxStop")))}return w},getJSON:function(a,b,c){return n.get(a,b,c,"json")},getScript:function(a,b){return n.get(a,void 0,b,"script")}}),n.each(["get","post"],function(a,b){n[b]=function(a,c,d,e){return n.isFunction(c)&&(e=e||d,d=c,c=void 0),n.ajax(n.extend({url:a,type:b,dataType:e,data:c,success:d},n.isPlainObject(a)&&a))}}),n._evalUrl=function(a){return n.ajax({url:a,type:"GET",dataType:"script",cache:!0,async:!1,global:!1,"throws":!0})},n.fn.extend({wrapAll:function(a){if(n.isFunction(a))return this.each(function(b){n(this).wrapAll(a.call(this,b))});if(this[0]){var b=n(a,this[0].ownerDocument).eq(0).clone(!0);this[0].parentNode&&b.insertBefore(this[0]),b.map(function(){var a=this;while(a.firstChild&&1===a.firstChild.nodeType)a=a.firstChild;return a}).append(this)}return this},wrapInner:function(a){return n.isFunction(a)?this.each(function(b){n(this).wrapInner(a.call(this,b))}):this.each(function(){var b=n(this),c=b.contents();c.length?c.wrapAll(a):b.append(a)})},wrap:function(a){var b=n.isFunction(a);return this.each(function(c){n(this).wrapAll(b?a.call(this,c):a)})},unwrap:function(){return this.parent().each(function(){n.nodeName(this,"body")||n(this).replaceWith(this.childNodes)}).end()}});function Yb(a){return a.style&&a.style.display||n.css(a,"display")}function Zb(a){if(!n.contains(a.ownerDocument||d,a))return!0;while(a&&1===a.nodeType){if("none"===Yb(a)||"hidden"===a.type)return!0;a=a.parentNode}return!1}n.expr.filters.hidden=function(a){return l.reliableHiddenOffsets()?a.offsetWidth<=0&&a.offsetHeight<=0&&!a.getClientRects().length:Zb(a)},n.expr.filters.visible=function(a){return!n.expr.filters.hidden(a)};var $b=/%20/g,_b=/\[\]$/,ac=/\r?\n/g,bc=/^(?:submit|button|image|reset|file)$/i,cc=/^(?:input|select|textarea|keygen)/i;function dc(a,b,c,d){var e;if(n.isArray(b))n.each(b,function(b,e){c||_b.test(a)?d(a,e):dc(a+"["+("object"==typeof e&&null!=e?b:"")+"]",e,c,d)});else if(c||"object"!==n.type(b))d(a,b);else for(e in b)dc(a+"["+e+"]",b[e],c,d)}n.param=function(a,b){var c,d=[],e=function(a,b){b=n.isFunction(b)?b():null==b?"":b,d[d.length]=encodeURIComponent(a)+"="+encodeURIComponent(b)};if(void 0===b&&(b=n.ajaxSettings&&n.ajaxSettings.traditional),n.isArray(a)||a.jquery&&!n.isPlainObject(a))n.each(a,function(){e(this.name,this.value)});else for(c in a)dc(c,a[c],b,e);return d.join("&").replace($b,"+")},n.fn.extend({serialize:function(){return n.param(this.serializeArray())},serializeArray:function(){return this.map(function(){var a=n.prop(this,"elements");return a?n.makeArray(a):this}).filter(function(){var a=this.type;return this.name&&!n(this).is(":disabled")&&cc.test(this.nodeName)&&!bc.test(a)&&(this.checked||!Z.test(a))}).map(function(a,b){var c=n(this).val();return null==c?null:n.isArray(c)?n.map(c,function(a){return{name:b.name,value:a.replace(ac,"\r\n")}}):{name:b.name,value:c.replace(ac,"\r\n")}}).get()}}),n.ajaxSettings.xhr=void 0!==a.ActiveXObject?function(){return this.isLocal?ic():d.documentMode>8?hc():/^(get|post|head|put|delete|options)$/i.test(this.type)&&hc()||ic()}:hc;var ec=0,fc={},gc=n.ajaxSettings.xhr();a.attachEvent&&a.attachEvent("onunload",function(){for(var a in fc)fc[a](void 0,!0)}),l.cors=!!gc&&"withCredentials"in gc,gc=l.ajax=!!gc,gc&&n.ajaxTransport(function(b){if(!b.crossDomain||l.cors){var c;return{send:function(d,e){var f,g=b.xhr(),h=++ec;if(g.open(b.type,b.url,b.async,b.username,b.password),b.xhrFields)for(f in b.xhrFields)g[f]=b.xhrFields[f];b.mimeType&&g.overrideMimeType&&g.overrideMimeType(b.mimeType),b.crossDomain||d["X-Requested-With"]||(d["X-Requested-With"]="XMLHttpRequest");for(f in d)void 0!==d[f]&&g.setRequestHeader(f,d[f]+"");g.send(b.hasContent&&b.data||null),c=function(a,d){var f,i,j;if(c&&(d||4===g.readyState))if(delete fc[h],c=void 0,g.onreadystatechange=n.noop,d)4!==g.readyState&&g.abort();else{j={},f=g.status,"string"==typeof g.responseText&&(j.text=g.responseText);try{i=g.statusText}catch(k){i=""}f||!b.isLocal||b.crossDomain?1223===f&&(f=204):f=j.text?200:404}j&&e(f,i,j,g.getAllResponseHeaders())},b.async?4===g.readyState?a.setTimeout(c):g.onreadystatechange=fc[h]=c:c()},abort:function(){c&&c(void 0,!0)}}}});function hc(){try{return new a.XMLHttpRequest}catch(b){}}function ic(){try{return new a.ActiveXObject("Microsoft.XMLHTTP")}catch(b){}}n.ajaxSetup({accepts:{script:"text/javascript, application/javascript, application/ecmascript, application/x-ecmascript"},contents:{script:/\b(?:java|ecma)script\b/},converters:{"text script":function(a){return n.globalEval(a),a}}}),n.ajaxPrefilter("script",function(a){void 0===a.cache&&(a.cache=!1),a.crossDomain&&(a.type="GET",a.global=!1)}),n.ajaxTransport("script",function(a){if(a.crossDomain){var b,c=d.head||n("head")[0]||d.documentElement;return{send:function(e,f){b=d.createElement("script"),b.async=!0,a.scriptCharset&&(b.charset=a.scriptCharset),b.src=a.url,b.onload=b.onreadystatechange=function(a,c){(c||!b.readyState||/loaded|complete/.test(b.readyState))&&(b.onload=b.onreadystatechange=null,b.parentNode&&b.parentNode.removeChild(b),b=null,c||f(200,"success"))},c.insertBefore(b,c.firstChild)},abort:function(){b&&b.onload(void 0,!0)}}}});var jc=[],kc=/(=)\?(?=&|$)|\?\?/;n.ajaxSetup({jsonp:"callback",jsonpCallback:function(){var a=jc.pop()||n.expando+"_"+Eb++;return this[a]=!0,a}}),n.ajaxPrefilter("json jsonp",function(b,c,d){var e,f,g,h=b.jsonp!==!1&&(kc.test(b.url)?"url":"string"==typeof b.data&&0===(b.contentType||"").indexOf("application/x-www-form-urlencoded")&&kc.test(b.data)&&"data");return h||"jsonp"===b.dataTypes[0]?(e=b.jsonpCallback=n.isFunction(b.jsonpCallback)?b.jsonpCallback():b.jsonpCallback,h?b[h]=b[h].replace(kc,"$1"+e):b.jsonp!==!1&&(b.url+=(Fb.test(b.url)?"&":"?")+b.jsonp+"="+e),b.converters["script json"]=function(){return g||n.error(e+" was not called"),g[0]},b.dataTypes[0]="json",f=a[e],a[e]=function(){g=arguments},d.always(function(){void 0===f?n(a).removeProp(e):a[e]=f,b[e]&&(b.jsonpCallback=c.jsonpCallback,jc.push(e)),g&&n.isFunction(f)&&f(g[0]),g=f=void 0}),"script"):void 0}),n.parseHTML=function(a,b,c){if(!a||"string"!=typeof a)return null;"boolean"==typeof b&&(c=b,b=!1),b=b||d;var e=x.exec(a),f=!c&&[];return e?[b.createElement(e[1])]:(e=ja([a],b,f),f&&f.length&&n(f).remove(),n.merge([],e.childNodes))};var lc=n.fn.load;n.fn.load=function(a,b,c){if("string"!=typeof a&&lc)return lc.apply(this,arguments);var d,e,f,g=this,h=a.indexOf(" ");return h>-1&&(d=n.trim(a.slice(h,a.length)),a=a.slice(0,h)),n.isFunction(b)?(c=b,b=void 0):b&&"object"==typeof b&&(e="POST"),g.length>0&&n.ajax({url:a,type:e||"GET",dataType:"html",data:b}).done(function(a){f=arguments,g.html(d?n("<div>").append(n.parseHTML(a)).find(d):a)}).always(c&&function(a,b){g.each(function(){c.apply(this,f||[a.responseText,b,a])})}),this},n.each(["ajaxStart","ajaxStop","ajaxComplete","ajaxError","ajaxSuccess","ajaxSend"],function(a,b){n.fn[b]=function(a){return this.on(b,a)}}),n.expr.filters.animated=function(a){return n.grep(n.timers,function(b){return a===b.elem}).length};function mc(a){return n.isWindow(a)?a:9===a.nodeType?a.defaultView||a.parentWindow:!1}n.offset={setOffset:function(a,b,c){var d,e,f,g,h,i,j,k=n.css(a,"position"),l=n(a),m={};"static"===k&&(a.style.position="relative"),h=l.offset(),f=n.css(a,"top"),i=n.css(a,"left"),j=("absolute"===k||"fixed"===k)&&n.inArray("auto",[f,i])>-1,j?(d=l.position(),g=d.top,e=d.left):(g=parseFloat(f)||0,e=parseFloat(i)||0),n.isFunction(b)&&(b=b.call(a,c,n.extend({},h))),null!=b.top&&(m.top=b.top-h.top+g),null!=b.left&&(m.left=b.left-h.left+e),"using"in b?b.using.call(a,m):l.css(m)}},n.fn.extend({offset:function(a){if(arguments.length)return void 0===a?this:this.each(function(b){n.offset.setOffset(this,a,b)});var b,c,d={top:0,left:0},e=this[0],f=e&&e.ownerDocument;if(f)return b=f.documentElement,n.contains(b,e)?("undefined"!=typeof e.getBoundingClientRect&&(d=e.getBoundingClientRect()),c=mc(f),{top:d.top+(c.pageYOffset||b.scrollTop)-(b.clientTop||0),left:d.left+(c.pageXOffset||b.scrollLeft)-(b.clientLeft||0)}):d},position:function(){if(this[0]){var a,b,c={top:0,left:0},d=this[0];return"fixed"===n.css(d,"position")?b=d.getBoundingClientRect():(a=this.offsetParent(),b=this.offset(),n.nodeName(a[0],"html")||(c=a.offset()),c.top+=n.css(a[0],"borderTopWidth",!0),c.left+=n.css(a[0],"borderLeftWidth",!0)),{top:b.top-c.top-n.css(d,"marginTop",!0),left:b.left-c.left-n.css(d,"marginLeft",!0)}}},offsetParent:function(){return this.map(function(){var a=this.offsetParent;while(a&&!n.nodeName(a,"html")&&"static"===n.css(a,"position"))a=a.offsetParent;return a||Qa})}}),n.each({scrollLeft:"pageXOffset",scrollTop:"pageYOffset"},function(a,b){var c=/Y/.test(b);n.fn[a]=function(d){return Y(this,function(a,d,e){var f=mc(a);return void 0===e?f?b in f?f[b]:f.document.documentElement[d]:a[d]:void(f?f.scrollTo(c?n(f).scrollLeft():e,c?e:n(f).scrollTop()):a[d]=e)},a,d,arguments.length,null)}}),n.each(["top","left"],function(a,b){n.cssHooks[b]=Ua(l.pixelPosition,function(a,c){return c?(c=Sa(a,b),Oa.test(c)?n(a).position()[b]+"px":c):void 0})}),n.each({Height:"height",Width:"width"},function(a,b){n.each({
padding:"inner"+a,content:b,"":"outer"+a},function(c,d){n.fn[d]=function(d,e){var f=arguments.length&&(c||"boolean"!=typeof d),g=c||(d===!0||e===!0?"margin":"border");return Y(this,function(b,c,d){var e;return n.isWindow(b)?b.document.documentElement["client"+a]:9===b.nodeType?(e=b.documentElement,Math.max(b.body["scroll"+a],e["scroll"+a],b.body["offset"+a],e["offset"+a],e["client"+a])):void 0===d?n.css(b,c,g):n.style(b,c,d,g)},b,f?d:void 0,f,null)}})}),n.fn.extend({bind:function(a,b,c){return this.on(a,null,b,c)},unbind:function(a,b){return this.off(a,null,b)},delegate:function(a,b,c,d){return this.on(b,a,c,d)},undelegate:function(a,b,c){return 1===arguments.length?this.off(a,"**"):this.off(b,a||"**",c)}}),n.fn.size=function(){return this.length},n.fn.andSelf=n.fn.addBack,"function"==typeof define&&define.amd&&define("jquery",[],function(){return n});var nc=a.jQuery,oc=a.$;return n.noConflict=function(b){return a.$===n&&(a.$=oc),b&&a.jQuery===n&&(a.jQuery=nc),n},b||(a.jQuery=a.$=n),n});
</script>
<style type="text/css">.leaflet-pane,.leaflet-tile,.leaflet-marker-icon,.leaflet-marker-shadow,.leaflet-tile-container,.leaflet-pane > svg,.leaflet-pane > canvas,.leaflet-zoom-box,.leaflet-image-layer,.leaflet-layer {position: absolute;left: 0;top: 0;}.leaflet-container {overflow: hidden;}.leaflet-tile,.leaflet-marker-icon,.leaflet-marker-shadow {-webkit-user-select: none;-moz-user-select: none;user-select: none;-webkit-user-drag: none;}.leaflet-safari .leaflet-tile {image-rendering: -webkit-optimize-contrast;}.leaflet-safari .leaflet-tile-container {width: 1600px;height: 1600px;-webkit-transform-origin: 0 0;}.leaflet-marker-icon,.leaflet-marker-shadow {display: block;}.leaflet-container .leaflet-overlay-pane svg,.leaflet-container .leaflet-marker-pane img,.leaflet-container .leaflet-shadow-pane img,.leaflet-container .leaflet-tile-pane img,.leaflet-container img.leaflet-image-layer {max-width: none !important;max-height: none !important;}.leaflet-container.leaflet-touch-zoom {-ms-touch-action: pan-x pan-y;touch-action: pan-x pan-y;}.leaflet-container.leaflet-touch-drag {-ms-touch-action: pinch-zoom;touch-action: none;touch-action: pinch-zoom;}.leaflet-container.leaflet-touch-drag.leaflet-touch-zoom {-ms-touch-action: none;touch-action: none;}.leaflet-container {-webkit-tap-highlight-color: transparent;}.leaflet-container a {-webkit-tap-highlight-color: rgba(51, 181, 229, 0.4);}.leaflet-tile {filter: inherit;visibility: hidden;}.leaflet-tile-loaded {visibility: inherit;}.leaflet-zoom-box {width: 0;height: 0;-moz-box-sizing: border-box;box-sizing: border-box;z-index: 800;}.leaflet-overlay-pane svg {-moz-user-select: none;}.leaflet-pane { z-index: 400; }.leaflet-tile-pane { z-index: 200; }.leaflet-overlay-pane { z-index: 400; }.leaflet-shadow-pane { z-index: 500; }.leaflet-marker-pane { z-index: 600; }.leaflet-tooltip-pane { z-index: 650; }.leaflet-popup-pane { z-index: 700; }.leaflet-map-pane canvas { z-index: 100; }.leaflet-map-pane svg { z-index: 200; }.leaflet-vml-shape {width: 1px;height: 1px;}.lvml {behavior: url(#default#VML);display: inline-block;position: absolute;}.leaflet-control {position: relative;z-index: 800;pointer-events: visiblePainted; pointer-events: auto;}.leaflet-top,.leaflet-bottom {position: absolute;z-index: 1000;pointer-events: none;}.leaflet-top {top: 0;}.leaflet-right {right: 0;}.leaflet-bottom {bottom: 0;}.leaflet-left {left: 0;}.leaflet-control {float: left;clear: both;}.leaflet-right .leaflet-control {float: right;}.leaflet-top .leaflet-control {margin-top: 10px;}.leaflet-bottom .leaflet-control {margin-bottom: 10px;}.leaflet-left .leaflet-control {margin-left: 10px;}.leaflet-right .leaflet-control {margin-right: 10px;}.leaflet-fade-anim .leaflet-tile {will-change: opacity;}.leaflet-fade-anim .leaflet-popup {opacity: 0;-webkit-transition: opacity 0.2s linear;-moz-transition: opacity 0.2s linear;-o-transition: opacity 0.2s linear;transition: opacity 0.2s linear;}.leaflet-fade-anim .leaflet-map-pane .leaflet-popup {opacity: 1;}.leaflet-zoom-animated {-webkit-transform-origin: 0 0;-ms-transform-origin: 0 0;transform-origin: 0 0;}.leaflet-zoom-anim .leaflet-zoom-animated {will-change: transform;}.leaflet-zoom-anim .leaflet-zoom-animated {-webkit-transition: -webkit-transform 0.25s cubic-bezier(0,0,0.25,1);-moz-transition: -moz-transform 0.25s cubic-bezier(0,0,0.25,1);-o-transition: -o-transform 0.25s cubic-bezier(0,0,0.25,1);transition: transform 0.25s cubic-bezier(0,0,0.25,1);}.leaflet-zoom-anim .leaflet-tile,.leaflet-pan-anim .leaflet-tile {-webkit-transition: none;-moz-transition: none;-o-transition: none;transition: none;}.leaflet-zoom-anim .leaflet-zoom-hide {visibility: hidden;}.leaflet-interactive {cursor: pointer;}.leaflet-grab {cursor: -webkit-grab;cursor: -moz-grab;}.leaflet-crosshair,.leaflet-crosshair .leaflet-interactive {cursor: crosshair;}.leaflet-popup-pane,.leaflet-control {cursor: auto;}.leaflet-dragging .leaflet-grab,.leaflet-dragging .leaflet-grab .leaflet-interactive,.leaflet-dragging .leaflet-marker-draggable {cursor: move;cursor: -webkit-grabbing;cursor: -moz-grabbing;}.leaflet-marker-icon,.leaflet-marker-shadow,.leaflet-image-layer,.leaflet-pane > svg path,.leaflet-tile-container {pointer-events: none;}.leaflet-marker-icon.leaflet-interactive,.leaflet-image-layer.leaflet-interactive,.leaflet-pane > svg path.leaflet-interactive {pointer-events: visiblePainted; pointer-events: auto;}.leaflet-container {background: #ddd;outline: 0;}.leaflet-container a {color: #0078A8;}.leaflet-container a.leaflet-active {outline: 2px solid orange;}.leaflet-zoom-box {border: 2px dotted #38f;background: rgba(255,255,255,0.5);}.leaflet-container {font: 12px/1.5 "Helvetica Neue", Arial, Helvetica, sans-serif;}.leaflet-bar {box-shadow: 0 1px 5px rgba(0,0,0,0.65);border-radius: 4px;}.leaflet-bar a,.leaflet-bar a:hover {background-color: #fff;border-bottom: 1px solid #ccc;width: 26px;height: 26px;line-height: 26px;display: block;text-align: center;text-decoration: none;color: black;}.leaflet-bar a,.leaflet-control-layers-toggle {background-position: 50% 50%;background-repeat: no-repeat;display: block;}.leaflet-bar a:hover {background-color: #f4f4f4;}.leaflet-bar a:first-child {border-top-left-radius: 4px;border-top-right-radius: 4px;}.leaflet-bar a:last-child {border-bottom-left-radius: 4px;border-bottom-right-radius: 4px;border-bottom: none;}.leaflet-bar a.leaflet-disabled {cursor: default;background-color: #f4f4f4;color: #bbb;}.leaflet-touch .leaflet-bar a {width: 30px;height: 30px;line-height: 30px;}.leaflet-touch .leaflet-bar a:first-child {border-top-left-radius: 2px;border-top-right-radius: 2px;}.leaflet-touch .leaflet-bar a:last-child {border-bottom-left-radius: 2px;border-bottom-right-radius: 2px;}.leaflet-control-zoom-in,.leaflet-control-zoom-out {font: bold 18px 'Lucida Console', Monaco, monospace;text-indent: 1px;}.leaflet-touch .leaflet-control-zoom-in, .leaflet-touch .leaflet-control-zoom-out {font-size: 22px;}.leaflet-control-layers {box-shadow: 0 1px 5px rgba(0,0,0,0.4);background: #fff;border-radius: 5px;}.leaflet-control-layers-toggle {background-image: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABoAAAAaCAQAAAADQ4RFAAACf0lEQVR4AY1UM3gkARTePdvdoTxXKc+qTl3aU5U6b2Kbkz3Gtq3Zw6ziLGNPzrYx7946Tr6/ee/XeCQ4D3ykPtL5tHno4n0d/h3+xfuWHGLX81cn7r0iTNzjr7LrlxCqPtkbTQEHeqOrTy4Yyt3VCi/IOB0v7rVC7q45Q3Gr5K6jt+3Gl5nCoDD4MtO+j96Wu8atmhGqcNGHObuf8OM/x3AMx38+4Z2sPqzCxRFK2aF2e5Jol56XTLyggAMTL56XOMoS1W4pOyjUcGGQdZxU6qRh7B9Zp+PfpOFlqt0zyDZckPi1ttmIp03jX8gyJ8a/PG2yutpS/Vol7peZIbZcKBAEEheEIAgFbDkz5H6Zrkm2hVWGiXKiF4Ycw0RWKdtC16Q7qe3X4iOMxruonzegJzWaXFrU9utOSsLUmrc0YjeWYjCW4PDMADElpJSSQ0vQvA1Tm6/JlKnqFs1EGyZiFCqnRZTEJJJiKRYzVYzJck2Rm6P4iH+cmSY0YzimYa8l0EtTODFWhcMIMVqdsI2uiTvKmTisIDHJ3od5GILVhBCarCfVRmo4uTjkhrhzkiBV7SsaqS+TzrzM1qpGGUFt28pIySQHR6h7F6KSwGWm97ay+Z+ZqMcEjEWebE7wxCSQwpkhJqoZA5ivCdZDjJepuJ9IQjGGUmuXJdBFUygxVqVsxFsLMbDe8ZbDYVCGKxs+W080max1hFCarCfV+C1KATwcnvE9gRRuMP2prdbWGowm1KB1y+zwMMENkM755cJ2yPDtqhTI6ED1M/82yIDtC/4j4BijjeObflpO9I9MwXTCsSX8jWAFeHr05WoLTJ5G8IQVS/7vwR6ohirYM7f6HzYpogfS3R2OAAAAAElFTkSuQmCC);width: 36px;height: 36px;}.leaflet-retina .leaflet-control-layers-toggle {background-image: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADQAAAA0CAQAAABvcdNgAAAEsklEQVR4AWL4TydIhpZK1kpWOlg0w3ZXP6D2soBtG42jeI6ZmQTHzAxiTbSJsYLjO9HhP+WOmcuhciVnmHVQcJnp7DFvScowZorad/+V/fVzMdMT2g9Cv9guXGv/7pYOrXh2U+RRR3dSd9JRx6bIFc/ekqHI29JC6pJ5ZEh1yWkhkbcFeSjxgx3L2m1cb1C7bceyxA+CNjT/Ifff+/kDk2u/w/33/IeCMOSaWZ4glosqT3DNnNZQ7Cs58/3Ce5HL78iZH/vKVIaYlqzfdLu8Vi7dnvUbEza5Idt36tquZFldl6N5Z/POLof0XLK61mZCmJSWjVF9tEjUluu74IUXvgttuVIHE7YxSkaYhJZam7yiM9Pv82JYfl9nptxZaxMJE4YSPty+vF0+Y2up9d3wwijfjZbabqm/3bZ9ecKHsiGmRflnn1MW4pjHf9oLufyn2z3y1D6n8g8TZhxyzipLNPnAUpsOiuWimg52psrTZYnOWYNDTMuWBWa0tJb4rgq1UvmutpaYEbZlwU3CLJm/ayYjHW5/h7xWLn9Hh1vepDkyf7dE7MtT5LR4e7yYpHrkhOUpEfssBLq2pPhAqoSWKUkk7EDqkmK6RrCEzqDjhNDWNE+XSMvkJRDWlZTmCW0l0PHQGRZY5t1L83kT0Y3l2SItk5JAWHl2dCOBm+fPu3fo5/3v61RMCO9Jx2EEYYhb0rmNQMX/vm7gqOEJLcXTGw3CAuRNeyaPWwjR8PRqKQ1PDA/dpv+on9Shox52WFnx0KY8onHayrJzm87i5h9xGw/tfkev0jGsQizqezUKjk12hBMKJ4kbCqGPVNXudyyrShovGw5CgxsRICxF6aRmSjlBnHRzg7Gx8fKqEubI2rahQYdR1YgDIRQO7JvQyD52hoIQx0mxa0ODtW2Iozn1le2iIRdzwWewedyZzewidueOGqlsn1MvcnQpuVwLGG3/IR1hIKxCjelIDZ8ldqWz25jWAsnldEnK0Zxro19TGVb2ffIZEsIO89EIEDvKMPrzmBOQcKQ+rroye6NgRRxqR4U8EAkz0CL6uSGOm6KQCdWjvjRiSP1BPalCRS5iQYiEIvxuBMJEWgzSoHADcVMuN7IuqqTeyUPq22qFimFtxDyBBJEwNyt6TM88blFHao/6tWWhuuOM4SAK4EI4QmFHA+SEyWlp4EQoJ13cYGzMu7yszEIBOm2rVmHUNqwAIQabISNMRstmdhNWcFLsSm+0tjJH1MdRxO5Nx0WDMhCtgD6OKgZeljJqJKc9po8juskR9XN0Y1lZ3mWjLR9JCO1jRDMd0fpYC2VnvjBSEFg7wBENc0R9HFlb0xvF1+TBEpF68d+DHR6IOWVv2BECtxo46hOFUBd/APU57WIoEwJhIi2CdpyZX0m93BZicktMj1AS9dClteUFAUNUIEygRZCtik5zSxI9MubTBH1GOiHsiLJ3OCoSZkILa9PxiN0EbvhsAo8tdAf9Seepd36lGWHmtNANTv5Jd0z4QYyeo/UEJqxKRpg5LZx6btLPsOaEmdMyxYdlc8LMaJnikDlhclqmPiQnTEpLUIZEwkRagjYkEibQErwhkTAKCLQEbUgkzJQWc/0PstHHcfEdQ+UAAAAASUVORK5CYII=);background-size: 26px 26px;}.leaflet-touch .leaflet-control-layers-toggle {width: 44px;height: 44px;}.leaflet-control-layers .leaflet-control-layers-list,.leaflet-control-layers-expanded .leaflet-control-layers-toggle {display: none;}.leaflet-control-layers-expanded .leaflet-control-layers-list {display: block;position: relative;}.leaflet-control-layers-expanded {padding: 6px 10px 6px 6px;color: #333;background: #fff;}.leaflet-control-layers-scrollbar {overflow-y: scroll;overflow-x: hidden;padding-right: 5px;}.leaflet-control-layers-selector {margin-top: 2px;position: relative;top: 1px;}.leaflet-control-layers label {display: block;}.leaflet-control-layers-separator {height: 0;border-top: 1px solid #ddd;margin: 5px -10px 5px -6px;}.leaflet-default-icon-path {background-image: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABkAAAApCAYAAADAk4LOAAAFgUlEQVR4Aa1XA5BjWRTN2oW17d3YaZtr2962HUzbDNpjszW24mRt28p47v7zq/bXZtrp/lWnXr337j3nPCe85NcypgSFdugCpW5YoDAMRaIMqRi6aKq5E3YqDQO3qAwjVWrD8Ncq/RBpykd8oZUb/kaJutow8r1aP9II0WmLKLIsJyv1w/kqw9Ch2MYdB++12Onxee/QMwvf4/Dk/Lfp/i4nxTXtOoQ4pW5Aj7wpici1A9erdAN2OH64x8OSP9j3Ft3b7aWkTg/Fm91siTra0f9on5sQr9INejH6CUUUpavjFNq1B+Oadhxmnfa8RfEmN8VNAsQhPqF55xHkMzz3jSmChWU6f7/XZKNH+9+hBLOHYozuKQPxyMPUKkrX/K0uWnfFaJGS1QPRtZsOPtr3NsW0uyh6NNCOkU3Yz+bXbT3I8G3xE5EXLXtCXbbqwCO9zPQYPRTZ5vIDXD7U+w7rFDEoUUf7ibHIR4y6bLVPXrz8JVZEql13trxwue/uDivd3fkWRbS6/IA2bID4uk0UpF1N8qLlbBlXs4Ee7HLTfV1j54APvODnSfOWBqtKVvjgLKzF5YdEk5ewRkGlK0i33Eofffc7HT56jD7/6U+qH3Cx7SBLNntH5YIPvODnyfIXZYRVDPqgHtLs5ABHD3YzLuespb7t79FY34DjMwrVrcTuwlT55YMPvOBnRrJ4VXTdNnYug5ucHLBjEpt30701A3Ts+HEa73u6dT3FNWwflY86eMHPk+Yu+i6pzUpRrW7SNDg5JHR4KapmM5Wv2E8Tfcb1HoqqHMHU+uWDD7zg54mz5/2BSnizi9T1Dg4QQXLToGNCkb6tb1NU+QAlGr1++eADrzhn/u8Q2YZhQVlZ5+CAOtqfbhmaUCS1ezNFVm2imDbPmPng5wmz+gwh+oHDce0eUtQ6OGDIyR0uUhUsoO3vfDmmgOezH0mZN59x7MBi++WDL1g/eEiU3avlidO671bkLfwbw5XV2P8Pzo0ydy4t2/0eu33xYSOMOD8hTf4CrBtGMSoXfPLchX+J0ruSePw3LZeK0juPJbYzrhkH0io7B3k164hiGvawhOKMLkrQLyVpZg8rHFW7E2uHOL888IBPlNZ1FPzstSJM694fWr6RwpvcJK60+0HCILTBzZLFNdtAzJaohze60T8qBzyh5ZuOg5e7uwQppofEmf2++DYvmySqGBuKaicF1blQjhuHdvCIMvp8whTTfZzI7RldpwtSzL+F1+wkdZ2TBOW2gIF88PBTzD/gpeREAMEbxnJcaJHNHrpzji0gQCS6hdkEeYt9DF/2qPcEC8RM28Hwmr3sdNyht00byAut2k3gufWNtgtOEOFGUwcXWNDbdNbpgBGxEvKkOQsxivJx33iow0Vw5S6SVTrpVq11ysA2Rp7gTfPfktc6zhtXBBC+adRLshf6sG2RfHPZ5EAc4sVZ83yCN00Fk/4kggu40ZTvIEm5g24qtU4KjBrx/BTTH8ifVASAG7gKrnWxJDcU7x8X6Ecczhm3o6YicvsLXWfh3Ch1W0k8x0nXF+0fFxgt4phz8QvypiwCCFKMqXCnqXExjq10beH+UUA7+nG6mdG/Pu0f3LgFcGrl2s0kNNjpmoJ9o4B29CMO8dMT4Q5ox8uitF6fqsrJOr8qnwNbRzv6hSnG5wP+64C7h9lp30hKNtKdWjtdkbuPA19nJ7Tz3zR/ibgARbhb4AlhavcBebmTHcFl2fvYEnW0ox9xMxKBS8btJ+KiEbq9zA4RthQXDhPa0T9TEe69gWupwc6uBUphquXgf+/FrIjweHQS4/pduMe5ERUMHUd9xv8ZR98CxkS4F2n3EUrUZ10EYNw7BWm9x1GiPssi3GgiGRDKWRYZfXlON+dfNbM+GgIwYdwAAAAASUVORK5CYII=);}.leaflet-container .leaflet-control-attribution {background: #fff;background: rgba(255, 255, 255, 0.7);margin: 0;}.leaflet-control-attribution,.leaflet-control-scale-line {padding: 0 5px;color: #333;}.leaflet-control-attribution a {text-decoration: none;}.leaflet-control-attribution a:hover {text-decoration: underline;}.leaflet-container .leaflet-control-attribution,.leaflet-container .leaflet-control-scale {font-size: 11px;}.leaflet-left .leaflet-control-scale {margin-left: 5px;}.leaflet-bottom .leaflet-control-scale {margin-bottom: 5px;}.leaflet-control-scale-line {border: 2px solid #777;border-top: none;line-height: 1.1;padding: 2px 5px 1px;font-size: 11px;white-space: nowrap;overflow: hidden;-moz-box-sizing: border-box;box-sizing: border-box;background: #fff;background: rgba(255, 255, 255, 0.5);}.leaflet-control-scale-line:not(:first-child) {border-top: 2px solid #777;border-bottom: none;margin-top: -2px;}.leaflet-control-scale-line:not(:first-child):not(:last-child) {border-bottom: 2px solid #777;}.leaflet-touch .leaflet-control-attribution,.leaflet-touch .leaflet-control-layers,.leaflet-touch .leaflet-bar {box-shadow: none;}.leaflet-touch .leaflet-control-layers,.leaflet-touch .leaflet-bar {border: 2px solid rgba(0,0,0,0.2);background-clip: padding-box;}.leaflet-popup {position: absolute;text-align: center;margin-bottom: 20px;}.leaflet-popup-content-wrapper {padding: 1px;text-align: left;border-radius: 12px;}.leaflet-popup-content {margin: 13px 19px;line-height: 1.4;}.leaflet-popup-content p {margin: 18px 0;}.leaflet-popup-tip-container {width: 40px;height: 20px;position: absolute;left: 50%;margin-left: -20px;overflow: hidden;pointer-events: none;}.leaflet-popup-tip {width: 17px;height: 17px;padding: 1px;margin: -10px auto 0;-webkit-transform: rotate(45deg);-moz-transform: rotate(45deg);-ms-transform: rotate(45deg);-o-transform: rotate(45deg);transform: rotate(45deg);}.leaflet-popup-content-wrapper,.leaflet-popup-tip {background: white;color: #333;box-shadow: 0 3px 14px rgba(0,0,0,0.4);}.leaflet-container a.leaflet-popup-close-button {position: absolute;top: 0;right: 0;padding: 4px 4px 0 0;border: none;text-align: center;width: 18px;height: 14px;font: 16px/14px Tahoma, Verdana, sans-serif;color: #c3c3c3;text-decoration: none;font-weight: bold;background: transparent;}.leaflet-container a.leaflet-popup-close-button:hover {color: #999;}.leaflet-popup-scrolled {overflow: auto;border-bottom: 1px solid #ddd;border-top: 1px solid #ddd;}.leaflet-oldie .leaflet-popup-content-wrapper {zoom: 1;}.leaflet-oldie .leaflet-popup-tip {width: 24px;margin: 0 auto;-ms-filter: "progid:DXImageTransform.Microsoft.Matrix(M11=0.70710678, M12=0.70710678, M21=-0.70710678, M22=0.70710678)";filter: progid:DXImageTransform.Microsoft.Matrix(M11=0.70710678, M12=0.70710678, M21=-0.70710678, M22=0.70710678);}.leaflet-oldie .leaflet-popup-tip-container {margin-top: -1px;}.leaflet-oldie .leaflet-control-zoom,.leaflet-oldie .leaflet-control-layers,.leaflet-oldie .leaflet-popup-content-wrapper,.leaflet-oldie .leaflet-popup-tip {border: 1px solid #999;}.leaflet-div-icon {background: #fff;border: 1px solid #666;}.leaflet-tooltip {position: absolute;padding: 6px;background-color: #fff;border: 1px solid #fff;border-radius: 3px;color: #222;white-space: nowrap;-webkit-user-select: none;-moz-user-select: none;-ms-user-select: none;user-select: none;pointer-events: none;box-shadow: 0 1px 3px rgba(0,0,0,0.4);}.leaflet-tooltip.leaflet-clickable {cursor: pointer;pointer-events: auto;}.leaflet-tooltip-top:before,.leaflet-tooltip-bottom:before,.leaflet-tooltip-left:before,.leaflet-tooltip-right:before {position: absolute;pointer-events: none;border: 6px solid transparent;background: transparent;content: "";}.leaflet-tooltip-bottom {margin-top: 6px;}.leaflet-tooltip-top {margin-top: -6px;}.leaflet-tooltip-bottom:before,.leaflet-tooltip-top:before {left: 50%;margin-left: -6px;}.leaflet-tooltip-top:before {bottom: 0;margin-bottom: -12px;border-top-color: #fff;}.leaflet-tooltip-bottom:before {top: 0;margin-top: -12px;margin-left: -6px;border-bottom-color: #fff;}.leaflet-tooltip-left {margin-left: -6px;}.leaflet-tooltip-right {margin-left: 6px;}.leaflet-tooltip-left:before,.leaflet-tooltip-right:before {top: 50%;margin-top: -6px;}.leaflet-tooltip-left:before {right: 0;margin-right: -12px;border-left-color: #fff;}.leaflet-tooltip-right:before {left: 0;margin-left: -12px;border-right-color: #fff;}</style>
<script>/* @preserve
 * Leaflet 1.3.1+Detached: ba6f97fff8647e724e4dfe66d2ed7da11f908989.ba6f97f, a JS library for interactive maps. https://leafletjs.com
 * (c) 2010-2017 Vladimir Agafonkin, (c) 2010-2011 CloudMade
 */
!function(t,i){"object"==typeof exports&&"undefined"!=typeof module?i(exports):"function"==typeof define&&define.amd?define(["exports"],i):i(t.L={})}(this,function(t){"use strict";function i(t){var i,e,n,o;for(e=1,n=arguments.length;e<n;e++){o=arguments[e];for(i in o)t[i]=o[i]}return t}function e(t,i){var e=Array.prototype.slice;if(t.bind)return t.bind.apply(t,e.call(arguments,1));var n=e.call(arguments,2);return function(){return t.apply(i,n.length?n.concat(e.call(arguments)):arguments)}}function n(t){return t._leaflet_id=t._leaflet_id||++ti,t._leaflet_id}function o(t,i,e){var n,o,s,r;return r=function(){n=!1,o&&(s.apply(e,o),o=!1)},s=function(){n?o=arguments:(t.apply(e,arguments),setTimeout(r,i),n=!0)}}function s(t,i,e){var n=i[1],o=i[0],s=n-o;return t===n&&e?t:((t-o)%s+s)%s+o}function r(){return!1}function a(t,i){var e=Math.pow(10,void 0===i?6:i);return Math.round(t*e)/e}function h(t){return t.trim?t.trim():t.replace(/^\s+|\s+$/g,"")}function u(t){return h(t).split(/\s+/)}function l(t,i){t.hasOwnProperty("options")||(t.options=t.options?Qt(t.options):{});for(var e in i)t.options[e]=i[e];return t.options}function c(t,i,e){var n=[];for(var o in t)n.push(encodeURIComponent(e?o.toUpperCase():o)+"="+encodeURIComponent(t[o]));return(i&&-1!==i.indexOf("?")?"&":"?")+n.join("&")}function _(t,i){return t.replace(ii,function(t,e){var n=i[e];if(void 0===n)throw new Error("No value provided for variable "+t);return"function"==typeof n&&(n=n(i)),n})}function d(t,i){for(var e=0;e<t.length;e++)if(t[e]===i)return e;return-1}function p(t){return window["webkit"+t]||window["moz"+t]||window["ms"+t]}function m(t){var i=+new Date,e=Math.max(0,16-(i-oi));return oi=i+e,window.setTimeout(t,e)}function f(t,i,n){if(!n||si!==m)return si.call(window,e(t,i));t.call(i)}function g(t){t&&ri.call(window,t)}function v(){}function y(t){if("undefined"!=typeof L&&L&&L.Mixin){t=ei(t)?t:[t];for(var i=0;i<t.length;i++)t[i]===L.Mixin.Events&&console.warn("Deprecated include of L.Mixin.Events: this property will be removed in future releases, please inherit from L.Evented instead.",(new Error).stack)}}function x(t,i,e){this.x=e?Math.round(t):t,this.y=e?Math.round(i):i}function w(t,i,e){return t instanceof x?t:ei(t)?new x(t[0],t[1]):void 0===t||null===t?t:"object"==typeof t&&"x"in t&&"y"in t?new x(t.x,t.y):new x(t,i,e)}function P(t,i){if(t)for(var e=i?[t,i]:t,n=0,o=e.length;n<o;n++)this.extend(e[n])}function b(t,i){return!t||t instanceof P?t:new P(t,i)}function T(t,i){if(t)for(var e=i?[t,i]:t,n=0,o=e.length;n<o;n++)this.extend(e[n])}function z(t,i){return t instanceof T?t:new T(t,i)}function M(t,i,e){if(isNaN(t)||isNaN(i))throw new Error("Invalid LatLng object: ("+t+", "+i+")");this.lat=+t,this.lng=+i,void 0!==e&&(this.alt=+e)}function C(t,i,e){return t instanceof M?t:ei(t)&&"object"!=typeof t[0]?3===t.length?new M(t[0],t[1],t[2]):2===t.length?new M(t[0],t[1]):null:void 0===t||null===t?t:"object"==typeof t&&"lat"in t?new M(t.lat,"lng"in t?t.lng:t.lon,t.alt):void 0===i?null:new M(t,i,e)}function Z(t,i,e,n){if(ei(t))return this._a=t[0],this._b=t[1],this._c=t[2],void(this._d=t[3]);this._a=t,this._b=i,this._c=e,this._d=n}function S(t,i,e,n){return new Z(t,i,e,n)}function E(t){return document.createElementNS("http://www.w3.org/2000/svg",t)}function k(t,i){var e,n,o,s,r,a,h="";for(e=0,o=t.length;e<o;e++){for(n=0,s=(r=t[e]).length;n<s;n++)a=r[n],h+=(n?"L":"M")+a.x+" "+a.y;h+=i?Xi?"z":"x":""}return h||"M0 0"}function A(t){return navigator.userAgent.toLowerCase().indexOf(t)>=0}function I(t,i,e,n){return"touchstart"===i?O(t,e,n):"touchmove"===i?W(t,e,n):"touchend"===i&&H(t,e,n),this}function B(t,i,e){var n=t["_leaflet_"+i+e];return"touchstart"===i?t.removeEventListener(Qi,n,!1):"touchmove"===i?t.removeEventListener(te,n,!1):"touchend"===i&&(t.removeEventListener(ie,n,!1),t.removeEventListener(ee,n,!1)),this}function O(t,i,n){var o=e(function(t){if("mouse"!==t.pointerType&&t.MSPOINTER_TYPE_MOUSE&&t.pointerType!==t.MSPOINTER_TYPE_MOUSE){if(!(ne.indexOf(t.target.tagName)<0))return;$(t)}j(t,i)});t["_leaflet_touchstart"+n]=o,t.addEventListener(Qi,o,!1),se||(document.documentElement.addEventListener(Qi,R,!0),document.documentElement.addEventListener(te,D,!0),document.documentElement.addEventListener(ie,N,!0),document.documentElement.addEventListener(ee,N,!0),se=!0)}function R(t){oe[t.pointerId]=t,re++}function D(t){oe[t.pointerId]&&(oe[t.pointerId]=t)}function N(t){delete oe[t.pointerId],re--}function j(t,i){t.touches=[];for(var e in oe)t.touches.push(oe[e]);t.changedTouches=[t],i(t)}function W(t,i,e){var n=function(t){(t.pointerType!==t.MSPOINTER_TYPE_MOUSE&&"mouse"!==t.pointerType||0!==t.buttons)&&j(t,i)};t["_leaflet_touchmove"+e]=n,t.addEventListener(te,n,!1)}function H(t,i,e){var n=function(t){j(t,i)};t["_leaflet_touchend"+e]=n,t.addEventListener(ie,n,!1),t.addEventListener(ee,n,!1)}function F(t,i,e){function n(t){var i;if(Ui){if(!Pi||"mouse"===t.pointerType)return;i=re}else i=t.touches.length;if(!(i>1)){var e=Date.now(),n=e-(s||e);r=t.touches?t.touches[0]:t,a=n>0&&n<=h,s=e}}function o(t){if(a&&!r.cancelBubble){if(Ui){if(!Pi||"mouse"===t.pointerType)return;var e,n,o={};for(n in r)e=r[n],o[n]=e&&e.bind?e.bind(r):e;r=o}r.type="dblclick",i(r),s=null}}var s,r,a=!1,h=250;return t[ue+ae+e]=n,t[ue+he+e]=o,t[ue+"dblclick"+e]=i,t.addEventListener(ae,n,!1),t.addEventListener(he,o,!1),t.addEventListener("dblclick",i,!1),this}function U(t,i){var e=t[ue+ae+i],n=t[ue+he+i],o=t[ue+"dblclick"+i];return t.removeEventListener(ae,e,!1),t.removeEventListener(he,n,!1),Pi||t.removeEventListener("dblclick",o,!1),this}function V(t,i,e,n){if("object"==typeof i)for(var o in i)G(t,o,i[o],e);else for(var s=0,r=(i=u(i)).length;s<r;s++)G(t,i[s],e,n);return this}function q(t,i,e,n){if("object"==typeof i)for(var o in i)K(t,o,i[o],e);else if(i)for(var s=0,r=(i=u(i)).length;s<r;s++)K(t,i[s],e,n);else{for(var a in t[le])K(t,a,t[le][a]);delete t[le]}return this}function G(t,i,e,o){var s=i+n(e)+(o?"_"+n(o):"");if(t[le]&&t[le][s])return this;var r=function(i){return e.call(o||t,i||window.event)},a=r;Ui&&0===i.indexOf("touch")?I(t,i,r,s):!Vi||"dblclick"!==i||!F||Ui&&Si?"addEventListener"in t?"mousewheel"===i?t.addEventListener("onwheel"in t?"wheel":"mousewheel",r,!1):"mouseenter"===i||"mouseleave"===i?(r=function(i){i=i||window.event,ot(t,i)&&a(i)},t.addEventListener("mouseenter"===i?"mouseover":"mouseout",r,!1)):("click"===i&&Ti&&(r=function(t){st(t,a)}),t.addEventListener(i,r,!1)):"attachEvent"in t&&t.attachEvent("on"+i,r):F(t,r,s),t[le]=t[le]||{},t[le][s]=r}function K(t,i,e,o){var s=i+n(e)+(o?"_"+n(o):""),r=t[le]&&t[le][s];if(!r)return this;Ui&&0===i.indexOf("touch")?B(t,i,s):!Vi||"dblclick"!==i||!U||Ui&&Si?"removeEventListener"in t?"mousewheel"===i?t.removeEventListener("onwheel"in t?"wheel":"mousewheel",r,!1):t.removeEventListener("mouseenter"===i?"mouseover":"mouseleave"===i?"mouseout":i,r,!1):"detachEvent"in t&&t.detachEvent("on"+i,r):U(t,s),t[le][s]=null}function Y(t){return t.stopPropagation?t.stopPropagation():t.originalEvent?t.originalEvent._stopped=!0:t.cancelBubble=!0,nt(t),this}function X(t){return G(t,"mousewheel",Y),this}function J(t){return V(t,"mousedown touchstart dblclick",Y),G(t,"click",et),this}function $(t){return t.preventDefault?t.preventDefault():t.returnValue=!1,this}function Q(t){return $(t),Y(t),this}function tt(t,i){if(!i)return new x(t.clientX,t.clientY);var e=i.getBoundingClientRect(),n=e.width/i.offsetWidth||1,o=e.height/i.offsetHeight||1;return new x(t.clientX/n-e.left-i.clientLeft,t.clientY/o-e.top-i.clientTop)}function it(t){return Pi?t.wheelDeltaY/2:t.deltaY&&0===t.deltaMode?-t.deltaY/ce:t.deltaY&&1===t.deltaMode?20*-t.deltaY:t.deltaY&&2===t.deltaMode?60*-t.deltaY:t.deltaX||t.deltaZ?0:t.wheelDelta?(t.wheelDeltaY||t.wheelDelta)/2:t.detail&&Math.abs(t.detail)<32765?20*-t.detail:t.detail?t.detail/-32765*60:0}function et(t){_e[t.type]=!0}function nt(t){var i=_e[t.type];return _e[t.type]=!1,i}function ot(t,i){var e=i.relatedTarget;if(!e)return!0;try{for(;e&&e!==t;)e=e.parentNode}catch(t){return!1}return e!==t}function st(t,i){var e=t.timeStamp||t.originalEvent&&t.originalEvent.timeStamp,n=pi&&e-pi;n&&n>100&&n<500||t.target._simulatedClick&&!t._simulated?Q(t):(pi=e,i(t))}function rt(t){return"string"==typeof t?document.getElementById(t):t}function at(t,i){var e=t.style[i]||t.currentStyle&&t.currentStyle[i];if((!e||"auto"===e)&&document.defaultView){var n=document.defaultView.getComputedStyle(t,null);e=n?n[i]:null}return"auto"===e?null:e}function ht(t,i,e){var n=document.createElement(t);return n.className=i||"",e&&e.appendChild(n),n}function ut(t){var i=t.parentNode;i&&i.removeChild(t)}function lt(t){for(;t.firstChild;)t.removeChild(t.firstChild)}function ct(t){var i=t.parentNode;i.lastChild!==t&&i.appendChild(t)}function _t(t){var i=t.parentNode;i.firstChild!==t&&i.insertBefore(t,i.firstChild)}function dt(t,i){if(void 0!==t.classList)return t.classList.contains(i);var e=gt(t);return e.length>0&&new RegExp("(^|\\s)"+i+"(\\s|$)").test(e)}function pt(t,i){if(void 0!==t.classList)for(var e=u(i),n=0,o=e.length;n<o;n++)t.classList.add(e[n]);else if(!dt(t,i)){var s=gt(t);ft(t,(s?s+" ":"")+i)}}function mt(t,i){void 0!==t.classList?t.classList.remove(i):ft(t,h((" "+gt(t)+" ").replace(" "+i+" "," ")))}function ft(t,i){void 0===t.className.baseVal?t.className=i:t.className.baseVal=i}function gt(t){return void 0===t.className.baseVal?t.className:t.className.baseVal}function vt(t,i){"opacity"in t.style?t.style.opacity=i:"filter"in t.style&&yt(t,i)}function yt(t,i){var e=!1,n="DXImageTransform.Microsoft.Alpha";try{e=t.filters.item(n)}catch(t){if(1===i)return}i=Math.round(100*i),e?(e.Enabled=100!==i,e.Opacity=i):t.style.filter+=" progid:"+n+"(opacity="+i+")"}function xt(t){for(var i=document.documentElement.style,e=0;e<t.length;e++)if(t[e]in i)return t[e];return!1}function wt(t,i,e){var n=i||new x(0,0);t.style[pe]=(Oi?"translate("+n.x+"px,"+n.y+"px)":"translate3d("+n.x+"px,"+n.y+"px,0)")+(e?" scale("+e+")":"")}function Lt(t,i){t._leaflet_pos=i,Ni?wt(t,i):(t.style.left=i.x+"px",t.style.top=i.y+"px")}function Pt(t){return t._leaflet_pos||new x(0,0)}function bt(){V(window,"dragstart",$)}function Tt(){q(window,"dragstart",$)}function zt(t){for(;-1===t.tabIndex;)t=t.parentNode;t.style&&(Mt(),ve=t,ye=t.style.outline,t.style.outline="none",V(window,"keydown",Mt))}function Mt(){ve&&(ve.style.outline=ye,ve=void 0,ye=void 0,q(window,"keydown",Mt))}function Ct(t,i){if(!i||!t.length)return t.slice();var e=i*i;return t=kt(t,e),t=St(t,e)}function Zt(t,i,e){return Math.sqrt(Rt(t,i,e,!0))}function St(t,i){var e=t.length,n=new(typeof Uint8Array!=void 0+""?Uint8Array:Array)(e);n[0]=n[e-1]=1,Et(t,n,i,0,e-1);var o,s=[];for(o=0;o<e;o++)n[o]&&s.push(t[o]);return s}function Et(t,i,e,n,o){var s,r,a,h=0;for(r=n+1;r<=o-1;r++)(a=Rt(t[r],t[n],t[o],!0))>h&&(s=r,h=a);h>e&&(i[s]=1,Et(t,i,e,n,s),Et(t,i,e,s,o))}function kt(t,i){for(var e=[t[0]],n=1,o=0,s=t.length;n<s;n++)Ot(t[n],t[o])>i&&(e.push(t[n]),o=n);return o<s-1&&e.push(t[s-1]),e}function At(t,i,e,n,o){var s,r,a,h=n?Se:Bt(t,e),u=Bt(i,e);for(Se=u;;){if(!(h|u))return[t,i];if(h&u)return!1;a=Bt(r=It(t,i,s=h||u,e,o),e),s===h?(t=r,h=a):(i=r,u=a)}}function It(t,i,e,n,o){var s,r,a=i.x-t.x,h=i.y-t.y,u=n.min,l=n.max;return 8&e?(s=t.x+a*(l.y-t.y)/h,r=l.y):4&e?(s=t.x+a*(u.y-t.y)/h,r=u.y):2&e?(s=l.x,r=t.y+h*(l.x-t.x)/a):1&e&&(s=u.x,r=t.y+h*(u.x-t.x)/a),new x(s,r,o)}function Bt(t,i){var e=0;return t.x<i.min.x?e|=1:t.x>i.max.x&&(e|=2),t.y<i.min.y?e|=4:t.y>i.max.y&&(e|=8),e}function Ot(t,i){var e=i.x-t.x,n=i.y-t.y;return e*e+n*n}function Rt(t,i,e,n){var o,s=i.x,r=i.y,a=e.x-s,h=e.y-r,u=a*a+h*h;return u>0&&((o=((t.x-s)*a+(t.y-r)*h)/u)>1?(s=e.x,r=e.y):o>0&&(s+=a*o,r+=h*o)),a=t.x-s,h=t.y-r,n?a*a+h*h:new x(s,r)}function Dt(t){return!ei(t[0])||"object"!=typeof t[0][0]&&void 0!==t[0][0]}function Nt(t){return console.warn("Deprecated use of _flat, please use L.LineUtil.isFlat instead."),Dt(t)}function jt(t,i,e){var n,o,s,r,a,h,u,l,c,_=[1,4,2,8];for(o=0,u=t.length;o<u;o++)t[o]._code=Bt(t[o],i);for(r=0;r<4;r++){for(l=_[r],n=[],o=0,s=(u=t.length)-1;o<u;s=o++)a=t[o],h=t[s],a._code&l?h._code&l||((c=It(h,a,l,i,e))._code=Bt(c,i),n.push(c)):(h._code&l&&((c=It(h,a,l,i,e))._code=Bt(c,i),n.push(c)),n.push(a));t=n}return t}function Wt(t,i){var e,n,o,s,r="Feature"===t.type?t.geometry:t,a=r?r.coordinates:null,h=[],u=i&&i.pointToLayer,l=i&&i.coordsToLatLng||Ht;if(!a&&!r)return null;switch(r.type){case"Point":return e=l(a),u?u(t,e):new Xe(e);case"MultiPoint":for(o=0,s=a.length;o<s;o++)e=l(a[o]),h.push(u?u(t,e):new Xe(e));return new qe(h);case"LineString":case"MultiLineString":return n=Ft(a,"LineString"===r.type?0:1,l),new tn(n,i);case"Polygon":case"MultiPolygon":return n=Ft(a,"Polygon"===r.type?1:2,l),new en(n,i);case"GeometryCollection":for(o=0,s=r.geometries.length;o<s;o++){var c=Wt({geometry:r.geometries[o],type:"Feature",properties:t.properties},i);c&&h.push(c)}return new qe(h);default:throw new Error("Invalid GeoJSON object.")}}function Ht(t){return new M(t[1],t[0],t[2])}function Ft(t,i,e){for(var n,o=[],s=0,r=t.length;s<r;s++)n=i?Ft(t[s],i-1,e):(e||Ht)(t[s]),o.push(n);return o}function Ut(t,i){return i="number"==typeof i?i:6,void 0!==t.alt?[a(t.lng,i),a(t.lat,i),a(t.alt,i)]:[a(t.lng,i),a(t.lat,i)]}function Vt(t,i,e,n){for(var o=[],s=0,r=t.length;s<r;s++)o.push(i?Vt(t[s],i-1,e,n):Ut(t[s],n));return!i&&e&&o.push(o[0]),o}function qt(t,e){return t.feature?i({},t.feature,{geometry:e}):Gt(e)}function Gt(t){return"Feature"===t.type||"FeatureCollection"===t.type?t:{type:"Feature",properties:{},geometry:t}}function Kt(t,i){return new nn(t,i)}function Yt(t,i){return new dn(t,i)}function Xt(t){return Yi?new fn(t):null}function Jt(t){return Xi||Ji?new xn(t):null}var $t=Object.freeze;Object.freeze=function(t){return t};var Qt=Object.create||function(){function t(){}return function(i){return t.prototype=i,new t}}(),ti=0,ii=/\{ *([\w_-]+) *\}/g,ei=Array.isArray||function(t){return"[object Array]"===Object.prototype.toString.call(t)},ni="data:image/gif;base64,R0lGODlhAQABAAD/ACwAAAAAAQABAAACADs=",oi=0,si=window.requestAnimationFrame||p("RequestAnimationFrame")||m,ri=window.cancelAnimationFrame||p("CancelAnimationFrame")||p("CancelRequestAnimationFrame")||function(t){window.clearTimeout(t)},ai=(Object.freeze||Object)({freeze:$t,extend:i,create:Qt,bind:e,lastId:ti,stamp:n,throttle:o,wrapNum:s,falseFn:r,formatNum:a,trim:h,splitWords:u,setOptions:l,getParamString:c,template:_,isArray:ei,indexOf:d,emptyImageUrl:ni,requestFn:si,cancelFn:ri,requestAnimFrame:f,cancelAnimFrame:g});v.extend=function(t){var e=function(){this.initialize&&this.initialize.apply(this,arguments),this.callInitHooks()},n=e.__super__=this.prototype,o=Qt(n);o.constructor=e,e.prototype=o;for(var s in this)this.hasOwnProperty(s)&&"prototype"!==s&&"__super__"!==s&&(e[s]=this[s]);return t.statics&&(i(e,t.statics),delete t.statics),t.includes&&(y(t.includes),i.apply(null,[o].concat(t.includes)),delete t.includes),o.options&&(t.options=i(Qt(o.options),t.options)),i(o,t),o._initHooks=[],o.callInitHooks=function(){if(!this._initHooksCalled){n.callInitHooks&&n.callInitHooks.call(this),this._initHooksCalled=!0;for(var t=0,i=o._initHooks.length;t<i;t++)o._initHooks[t].call(this)}},e},v.include=function(t){return i(this.prototype,t),this},v.mergeOptions=function(t){return i(this.prototype.options,t),this},v.addInitHook=function(t){var i=Array.prototype.slice.call(arguments,1),e="function"==typeof t?t:function(){this[t].apply(this,i)};return this.prototype._initHooks=this.prototype._initHooks||[],this.prototype._initHooks.push(e),this};var hi={on:function(t,i,e){if("object"==typeof t)for(var n in t)this._on(n,t[n],i);else for(var o=0,s=(t=u(t)).length;o<s;o++)this._on(t[o],i,e);return this},off:function(t,i,e){if(t)if("object"==typeof t)for(var n in t)this._off(n,t[n],i);else for(var o=0,s=(t=u(t)).length;o<s;o++)this._off(t[o],i,e);else delete this._events;return this},_on:function(t,i,e){this._events=this._events||{};var n=this._events[t];n||(n=[],this._events[t]=n),e===this&&(e=void 0);for(var o={fn:i,ctx:e},s=n,r=0,a=s.length;r<a;r++)if(s[r].fn===i&&s[r].ctx===e)return;s.push(o)},_off:function(t,i,e){var n,o,s;if(this._events&&(n=this._events[t]))if(i){if(e===this&&(e=void 0),n)for(o=0,s=n.length;o<s;o++){var a=n[o];if(a.ctx===e&&a.fn===i)return a.fn=r,this._firingCount&&(this._events[t]=n=n.slice()),void n.splice(o,1)}}else{for(o=0,s=n.length;o<s;o++)n[o].fn=r;delete this._events[t]}},fire:function(t,e,n){if(!this.listens(t,n))return this;var o=i({},e,{type:t,target:this,sourceTarget:e&&e.sourceTarget||this});if(this._events){var s=this._events[t];if(s){this._firingCount=this._firingCount+1||1;for(var r=0,a=s.length;r<a;r++){var h=s[r];h.fn.call(h.ctx||this,o)}this._firingCount--}}return n&&this._propagateEvent(o),this},listens:function(t,i){var e=this._events&&this._events[t];if(e&&e.length)return!0;if(i)for(var n in this._eventParents)if(this._eventParents[n].listens(t,i))return!0;return!1},once:function(t,i,n){if("object"==typeof t){for(var o in t)this.once(o,t[o],i);return this}var s=e(function(){this.off(t,i,n).off(t,s,n)},this);return this.on(t,i,n).on(t,s,n)},addEventParent:function(t){return this._eventParents=this._eventParents||{},this._eventParents[n(t)]=t,this},removeEventParent:function(t){return this._eventParents&&delete this._eventParents[n(t)],this},_propagateEvent:function(t){for(var e in this._eventParents)this._eventParents[e].fire(t.type,i({layer:t.target,propagatedFrom:t.target},t),!0)}};hi.addEventListener=hi.on,hi.removeEventListener=hi.clearAllEventListeners=hi.off,hi.addOneTimeEventListener=hi.once,hi.fireEvent=hi.fire,hi.hasEventListeners=hi.listens;var ui=v.extend(hi),li=Math.trunc||function(t){return t>0?Math.floor(t):Math.ceil(t)};x.prototype={clone:function(){return new x(this.x,this.y)},add:function(t){return this.clone()._add(w(t))},_add:function(t){return this.x+=t.x,this.y+=t.y,this},subtract:function(t){return this.clone()._subtract(w(t))},_subtract:function(t){return this.x-=t.x,this.y-=t.y,this},divideBy:function(t){return this.clone()._divideBy(t)},_divideBy:function(t){return this.x/=t,this.y/=t,this},multiplyBy:function(t){return this.clone()._multiplyBy(t)},_multiplyBy:function(t){return this.x*=t,this.y*=t,this},scaleBy:function(t){return new x(this.x*t.x,this.y*t.y)},unscaleBy:function(t){return new x(this.x/t.x,this.y/t.y)},round:function(){return this.clone()._round()},_round:function(){return this.x=Math.round(this.x),this.y=Math.round(this.y),this},floor:function(){return this.clone()._floor()},_floor:function(){return this.x=Math.floor(this.x),this.y=Math.floor(this.y),this},ceil:function(){return this.clone()._ceil()},_ceil:function(){return this.x=Math.ceil(this.x),this.y=Math.ceil(this.y),this},trunc:function(){return this.clone()._trunc()},_trunc:function(){return this.x=li(this.x),this.y=li(this.y),this},distanceTo:function(t){var i=(t=w(t)).x-this.x,e=t.y-this.y;return Math.sqrt(i*i+e*e)},equals:function(t){return(t=w(t)).x===this.x&&t.y===this.y},contains:function(t){return t=w(t),Math.abs(t.x)<=Math.abs(this.x)&&Math.abs(t.y)<=Math.abs(this.y)},toString:function(){return"Point("+a(this.x)+", "+a(this.y)+")"}},P.prototype={extend:function(t){return t=w(t),this.min||this.max?(this.min.x=Math.min(t.x,this.min.x),this.max.x=Math.max(t.x,this.max.x),this.min.y=Math.min(t.y,this.min.y),this.max.y=Math.max(t.y,this.max.y)):(this.min=t.clone(),this.max=t.clone()),this},getCenter:function(t){return new x((this.min.x+this.max.x)/2,(this.min.y+this.max.y)/2,t)},getBottomLeft:function(){return new x(this.min.x,this.max.y)},getTopRight:function(){return new x(this.max.x,this.min.y)},getTopLeft:function(){return this.min},getBottomRight:function(){return this.max},getSize:function(){return this.max.subtract(this.min)},contains:function(t){var i,e;return(t="number"==typeof t[0]||t instanceof x?w(t):b(t))instanceof P?(i=t.min,e=t.max):i=e=t,i.x>=this.min.x&&e.x<=this.max.x&&i.y>=this.min.y&&e.y<=this.max.y},intersects:function(t){t=b(t);var i=this.min,e=this.max,n=t.min,o=t.max,s=o.x>=i.x&&n.x<=e.x,r=o.y>=i.y&&n.y<=e.y;return s&&r},overlaps:function(t){t=b(t);var i=this.min,e=this.max,n=t.min,o=t.max,s=o.x>i.x&&n.x<e.x,r=o.y>i.y&&n.y<e.y;return s&&r},isValid:function(){return!(!this.min||!this.max)}},T.prototype={extend:function(t){var i,e,n=this._southWest,o=this._northEast;if(t instanceof M)i=t,e=t;else{if(!(t instanceof T))return t?this.extend(C(t)||z(t)):this;if(i=t._southWest,e=t._northEast,!i||!e)return this}return n||o?(n.lat=Math.min(i.lat,n.lat),n.lng=Math.min(i.lng,n.lng),o.lat=Math.max(e.lat,o.lat),o.lng=Math.max(e.lng,o.lng)):(this._southWest=new M(i.lat,i.lng),this._northEast=new M(e.lat,e.lng)),this},pad:function(t){var i=this._southWest,e=this._northEast,n=Math.abs(i.lat-e.lat)*t,o=Math.abs(i.lng-e.lng)*t;return new T(new M(i.lat-n,i.lng-o),new M(e.lat+n,e.lng+o))},getCenter:function(){return new M((this._southWest.lat+this._northEast.lat)/2,(this._southWest.lng+this._northEast.lng)/2)},getSouthWest:function(){return this._southWest},getNorthEast:function(){return this._northEast},getNorthWest:function(){return new M(this.getNorth(),this.getWest())},getSouthEast:function(){return new M(this.getSouth(),this.getEast())},getWest:function(){return this._southWest.lng},getSouth:function(){return this._southWest.lat},getEast:function(){return this._northEast.lng},getNorth:function(){return this._northEast.lat},contains:function(t){t="number"==typeof t[0]||t instanceof M||"lat"in t?C(t):z(t);var i,e,n=this._southWest,o=this._northEast;return t instanceof T?(i=t.getSouthWest(),e=t.getNorthEast()):i=e=t,i.lat>=n.lat&&e.lat<=o.lat&&i.lng>=n.lng&&e.lng<=o.lng},intersects:function(t){t=z(t);var i=this._southWest,e=this._northEast,n=t.getSouthWest(),o=t.getNorthEast(),s=o.lat>=i.lat&&n.lat<=e.lat,r=o.lng>=i.lng&&n.lng<=e.lng;return s&&r},overlaps:function(t){t=z(t);var i=this._southWest,e=this._northEast,n=t.getSouthWest(),o=t.getNorthEast(),s=o.lat>i.lat&&n.lat<e.lat,r=o.lng>i.lng&&n.lng<e.lng;return s&&r},toBBoxString:function(){return[this.getWest(),this.getSouth(),this.getEast(),this.getNorth()].join(",")},equals:function(t,i){return!!t&&(t=z(t),this._southWest.equals(t.getSouthWest(),i)&&this._northEast.equals(t.getNorthEast(),i))},isValid:function(){return!(!this._southWest||!this._northEast)}},M.prototype={equals:function(t,i){return!!t&&(t=C(t),Math.max(Math.abs(this.lat-t.lat),Math.abs(this.lng-t.lng))<=(void 0===i?1e-9:i))},toString:function(t){return"LatLng("+a(this.lat,t)+", "+a(this.lng,t)+")"},distanceTo:function(t){return _i.distance(this,C(t))},wrap:function(){return _i.wrapLatLng(this)},toBounds:function(t){var i=180*t/40075017,e=i/Math.cos(Math.PI/180*this.lat);return z([this.lat-i,this.lng-e],[this.lat+i,this.lng+e])},clone:function(){return new M(this.lat,this.lng,this.alt)}};var ci={latLngToPoint:function(t,i){var e=this.projection.project(t),n=this.scale(i);return this.transformation._transform(e,n)},pointToLatLng:function(t,i){var e=this.scale(i),n=this.transformation.untransform(t,e);return this.projection.unproject(n)},project:function(t){return this.projection.project(t)},unproject:function(t){return this.projection.unproject(t)},scale:function(t){return 256*Math.pow(2,t)},zoom:function(t){return Math.log(t/256)/Math.LN2},getProjectedBounds:function(t){if(this.infinite)return null;var i=this.projection.bounds,e=this.scale(t);return new P(this.transformation.transform(i.min,e),this.transformation.transform(i.max,e))},infinite:!1,wrapLatLng:function(t){var i=this.wrapLng?s(t.lng,this.wrapLng,!0):t.lng;return new M(this.wrapLat?s(t.lat,this.wrapLat,!0):t.lat,i,t.alt)},wrapLatLngBounds:function(t){var i=t.getCenter(),e=this.wrapLatLng(i),n=i.lat-e.lat,o=i.lng-e.lng;if(0===n&&0===o)return t;var s=t.getSouthWest(),r=t.getNorthEast();return new T(new M(s.lat-n,s.lng-o),new M(r.lat-n,r.lng-o))}},_i=i({},ci,{wrapLng:[-180,180],R:6371e3,distance:function(t,i){var e=Math.PI/180,n=t.lat*e,o=i.lat*e,s=Math.sin((i.lat-t.lat)*e/2),r=Math.sin((i.lng-t.lng)*e/2),a=s*s+Math.cos(n)*Math.cos(o)*r*r,h=2*Math.atan2(Math.sqrt(a),Math.sqrt(1-a));return this.R*h}}),di={R:6378137,MAX_LATITUDE:85.0511287798,project:function(t){var i=Math.PI/180,e=this.MAX_LATITUDE,n=Math.max(Math.min(e,t.lat),-e),o=Math.sin(n*i);return new x(this.R*t.lng*i,this.R*Math.log((1+o)/(1-o))/2)},unproject:function(t){var i=180/Math.PI;return new M((2*Math.atan(Math.exp(t.y/this.R))-Math.PI/2)*i,t.x*i/this.R)},bounds:function(){var t=6378137*Math.PI;return new P([-t,-t],[t,t])}()};Z.prototype={transform:function(t,i){return this._transform(t.clone(),i)},_transform:function(t,i){return i=i||1,t.x=i*(this._a*t.x+this._b),t.y=i*(this._c*t.y+this._d),t},untransform:function(t,i){return i=i||1,new x((t.x/i-this._b)/this._a,(t.y/i-this._d)/this._c)}};var pi,mi,fi,gi,vi=i({},_i,{code:"EPSG:3857",projection:di,transformation:function(){var t=.5/(Math.PI*di.R);return S(t,.5,-t,.5)}()}),yi=i({},vi,{code:"EPSG:900913"}),xi=document.documentElement.style,wi="ActiveXObject"in window,Li=wi&&!document.addEventListener,Pi="msLaunchUri"in navigator&&!("documentMode"in document),bi=A("webkit"),Ti=A("android"),zi=A("android 2")||A("android 3"),Mi=parseInt(/WebKit\/([0-9]+)|$/.exec(navigator.userAgent)[1],10),Ci=Ti&&A("Google")&&Mi<537&&!("AudioNode"in window),Zi=!!window.opera,Si=A("chrome"),Ei=A("gecko")&&!bi&&!Zi&&!wi,ki=!Si&&A("safari"),Ai=A("phantom"),Ii="OTransition"in xi,Bi=0===navigator.platform.indexOf("Win"),Oi=wi&&"transition"in xi,Ri="WebKitCSSMatrix"in window&&"m11"in new window.WebKitCSSMatrix&&!zi,Di="MozPerspective"in xi,Ni=!window.L_DISABLE_3D&&(Oi||Ri||Di)&&!Ii&&!Ai,ji="undefined"!=typeof orientation||A("mobile"),Wi=ji&&bi,Hi=ji&&Ri,Fi=!window.PointerEvent&&window.MSPointerEvent,Ui=!(!window.PointerEvent&&!Fi),Vi=!window.L_NO_TOUCH&&(Ui||"ontouchstart"in window||window.DocumentTouch&&document instanceof window.DocumentTouch),qi=ji&&Zi,Gi=ji&&Ei,Ki=(window.devicePixelRatio||window.screen.deviceXDPI/window.screen.logicalXDPI)>1,Yi=!!document.createElement("canvas").getContext,Xi=!(!document.createElementNS||!E("svg").createSVGRect),Ji=!Xi&&function(){try{var t=document.createElement("div");t.innerHTML='<v:shape adj="1"/>';var i=t.firstChild;return i.style.behavior="url(#default#VML)",i&&"object"==typeof i.adj}catch(t){return!1}}(),$i=(Object.freeze||Object)({ie:wi,ielt9:Li,edge:Pi,webkit:bi,android:Ti,android23:zi,androidStock:Ci,opera:Zi,chrome:Si,gecko:Ei,safari:ki,phantom:Ai,opera12:Ii,win:Bi,ie3d:Oi,webkit3d:Ri,gecko3d:Di,any3d:Ni,mobile:ji,mobileWebkit:Wi,mobileWebkit3d:Hi,msPointer:Fi,pointer:Ui,touch:Vi,mobileOpera:qi,mobileGecko:Gi,retina:Ki,canvas:Yi,svg:Xi,vml:Ji}),Qi=Fi?"MSPointerDown":"pointerdown",te=Fi?"MSPointerMove":"pointermove",ie=Fi?"MSPointerUp":"pointerup",ee=Fi?"MSPointerCancel":"pointercancel",ne=["INPUT","SELECT","OPTION"],oe={},se=!1,re=0,ae=Fi?"MSPointerDown":Ui?"pointerdown":"touchstart",he=Fi?"MSPointerUp":Ui?"pointerup":"touchend",ue="_leaflet_",le="_leaflet_events",ce=Bi&&Si?2*window.devicePixelRatio:Ei?window.devicePixelRatio:1,_e={},de=(Object.freeze||Object)({on:V,off:q,stopPropagation:Y,disableScrollPropagation:X,disableClickPropagation:J,preventDefault:$,stop:Q,getMousePosition:tt,getWheelDelta:it,fakeStop:et,skipped:nt,isExternalTarget:ot,addListener:V,removeListener:q}),pe=xt(["transform","WebkitTransform","OTransform","MozTransform","msTransform"]),me=xt(["webkitTransition","transition","OTransition","MozTransition","msTransition"]),fe="webkitTransition"===me||"OTransition"===me?me+"End":"transitionend";if("onselectstart"in document)mi=function(){V(window,"selectstart",$)},fi=function(){q(window,"selectstart",$)};else{var ge=xt(["userSelect","WebkitUserSelect","OUserSelect","MozUserSelect","msUserSelect"]);mi=function(){if(ge){var t=document.documentElement.style;gi=t[ge],t[ge]="none"}},fi=function(){ge&&(document.documentElement.style[ge]=gi,gi=void 0)}}var ve,ye,xe=(Object.freeze||Object)({TRANSFORM:pe,TRANSITION:me,TRANSITION_END:fe,get:rt,getStyle:at,create:ht,remove:ut,empty:lt,toFront:ct,toBack:_t,hasClass:dt,addClass:pt,removeClass:mt,setClass:ft,getClass:gt,setOpacity:vt,testProp:xt,setTransform:wt,setPosition:Lt,getPosition:Pt,disableTextSelection:mi,enableTextSelection:fi,disableImageDrag:bt,enableImageDrag:Tt,preventOutline:zt,restoreOutline:Mt}),we=ui.extend({run:function(t,i,e,n){this.stop(),this._el=t,this._inProgress=!0,this._duration=e||.25,this._easeOutPower=1/Math.max(n||.5,.2),this._startPos=Pt(t),this._offset=i.subtract(this._startPos),this._startTime=+new Date,this.fire("start"),this._animate()},stop:function(){this._inProgress&&(this._step(!0),this._complete())},_animate:function(){this._animId=f(this._animate,this),this._step()},_step:function(t){var i=+new Date-this._startTime,e=1e3*this._duration;i<e?this._runFrame(this._easeOut(i/e),t):(this._runFrame(1),this._complete())},_runFrame:function(t,i){var e=this._startPos.add(this._offset.multiplyBy(t));i&&e._round(),Lt(this._el,e),this.fire("step")},_complete:function(){g(this._animId),this._inProgress=!1,this.fire("end")},_easeOut:function(t){return 1-Math.pow(1-t,this._easeOutPower)}}),Le=ui.extend({options:{crs:vi,center:void 0,zoom:void 0,minZoom:void 0,maxZoom:void 0,layers:[],maxBounds:void 0,renderer:void 0,zoomAnimation:!0,zoomAnimationThreshold:4,fadeAnimation:!0,markerZoomAnimation:!0,transform3DLimit:8388608,zoomSnap:1,zoomDelta:1,trackResize:!0},initialize:function(t,i){i=l(this,i),this._initContainer(t),this._initLayout(),this._onResize=e(this._onResize,this),this._initEvents(),i.maxBounds&&this.setMaxBounds(i.maxBounds),void 0!==i.zoom&&(this._zoom=this._limitZoom(i.zoom)),i.center&&void 0!==i.zoom&&this.setView(C(i.center),i.zoom,{reset:!0}),this._handlers=[],this._layers={},this._zoomBoundLayers={},this._sizeChanged=!0,this.callInitHooks(),this._zoomAnimated=me&&Ni&&!qi&&this.options.zoomAnimation,this._zoomAnimated&&(this._createAnimProxy(),V(this._proxy,fe,this._catchTransitionEnd,this)),this._addLayers(this.options.layers)},setView:function(t,e,n){return e=void 0===e?this._zoom:this._limitZoom(e),t=this._limitCenter(C(t),e,this.options.maxBounds),n=n||{},this._stop(),this._loaded&&!n.reset&&!0!==n&&(void 0!==n.animate&&(n.zoom=i({animate:n.animate},n.zoom),n.pan=i({animate:n.animate,duration:n.duration},n.pan)),this._zoom!==e?this._tryAnimatedZoom&&this._tryAnimatedZoom(t,e,n.zoom):this._tryAnimatedPan(t,n.pan))?(clearTimeout(this._sizeTimer),this):(this._resetView(t,e),this)},setZoom:function(t,i){return this._loaded?this.setView(this.getCenter(),t,{zoom:i}):(this._zoom=t,this)},zoomIn:function(t,i){return t=t||(Ni?this.options.zoomDelta:1),this.setZoom(this._zoom+t,i)},zoomOut:function(t,i){return t=t||(Ni?this.options.zoomDelta:1),this.setZoom(this._zoom-t,i)},setZoomAround:function(t,i,e){var n=this.getZoomScale(i),o=this.getSize().divideBy(2),s=(t instanceof x?t:this.latLngToContainerPoint(t)).subtract(o).multiplyBy(1-1/n),r=this.containerPointToLatLng(o.add(s));return this.setView(r,i,{zoom:e})},_getBoundsCenterZoom:function(t,i){i=i||{},t=t.getBounds?t.getBounds():z(t);var e=w(i.paddingTopLeft||i.padding||[0,0]),n=w(i.paddingBottomRight||i.padding||[0,0]),o=this.getBoundsZoom(t,!1,e.add(n));if((o="number"==typeof i.maxZoom?Math.min(i.maxZoom,o):o)===1/0)return{center:t.getCenter(),zoom:o};var s=n.subtract(e).divideBy(2),r=this.project(t.getSouthWest(),o),a=this.project(t.getNorthEast(),o);return{center:this.unproject(r.add(a).divideBy(2).add(s),o),zoom:o}},fitBounds:function(t,i){if(!(t=z(t)).isValid())throw new Error("Bounds are not valid.");var e=this._getBoundsCenterZoom(t,i);return this.setView(e.center,e.zoom,i)},fitWorld:function(t){return this.fitBounds([[-90,-180],[90,180]],t)},panTo:function(t,i){return this.setView(t,this._zoom,{pan:i})},panBy:function(t,i){if(t=w(t).round(),i=i||{},!t.x&&!t.y)return this.fire("moveend");if(!0!==i.animate&&!this.getSize().contains(t))return this._resetView(this.unproject(this.project(this.getCenter()).add(t)),this.getZoom()),this;if(this._panAnim||(this._panAnim=new we,this._panAnim.on({step:this._onPanTransitionStep,end:this._onPanTransitionEnd},this)),i.noMoveStart||this.fire("movestart"),!1!==i.animate){pt(this._mapPane,"leaflet-pan-anim");var e=this._getMapPanePos().subtract(t).round();this._panAnim.run(this._mapPane,e,i.duration||.25,i.easeLinearity)}else this._rawPanBy(t),this.fire("move").fire("moveend");return this},flyTo:function(t,i,e){function n(t){var i=(g*g-m*m+(t?-1:1)*x*x*v*v)/(2*(t?g:m)*x*v),e=Math.sqrt(i*i+1)-i;return e<1e-9?-18:Math.log(e)}function o(t){return(Math.exp(t)-Math.exp(-t))/2}function s(t){return(Math.exp(t)+Math.exp(-t))/2}function r(t){return o(t)/s(t)}function a(t){return m*(s(w)/s(w+y*t))}function h(t){return m*(s(w)*r(w+y*t)-o(w))/x}function u(t){return 1-Math.pow(1-t,1.5)}function l(){var e=(Date.now()-L)/b,n=u(e)*P;e<=1?(this._flyToFrame=f(l,this),this._move(this.unproject(c.add(_.subtract(c).multiplyBy(h(n)/v)),p),this.getScaleZoom(m/a(n),p),{flyTo:!0})):this._move(t,i)._moveEnd(!0)}if(!1===(e=e||{}).animate||!Ni)return this.setView(t,i,e);this._stop();var c=this.project(this.getCenter()),_=this.project(t),d=this.getSize(),p=this._zoom;t=C(t),i=void 0===i?p:i;var m=Math.max(d.x,d.y),g=m*this.getZoomScale(p,i),v=_.distanceTo(c)||1,y=1.42,x=y*y,w=n(0),L=Date.now(),P=(n(1)-w)/y,b=e.duration?1e3*e.duration:1e3*P*.8;return this._moveStart(!0,e.noMoveStart),l.call(this),this},flyToBounds:function(t,i){var e=this._getBoundsCenterZoom(t,i);return this.flyTo(e.center,e.zoom,i)},setMaxBounds:function(t){return(t=z(t)).isValid()?(this.options.maxBounds&&this.off("moveend",this._panInsideMaxBounds),this.options.maxBounds=t,this._loaded&&this._panInsideMaxBounds(),this.on("moveend",this._panInsideMaxBounds)):(this.options.maxBounds=null,this.off("moveend",this._panInsideMaxBounds))},setMinZoom:function(t){var i=this.options.minZoom;return this.options.minZoom=t,this._loaded&&i!==t&&(this.fire("zoomlevelschange"),this.getZoom()<this.options.minZoom)?this.setZoom(t):this},setMaxZoom:function(t){var i=this.options.maxZoom;return this.options.maxZoom=t,this._loaded&&i!==t&&(this.fire("zoomlevelschange"),this.getZoom()>this.options.maxZoom)?this.setZoom(t):this},panInsideBounds:function(t,i){this._enforcingBounds=!0;var e=this.getCenter(),n=this._limitCenter(e,this._zoom,z(t));return e.equals(n)||this.panTo(n,i),this._enforcingBounds=!1,this},invalidateSize:function(t){if(!this._loaded)return this;t=i({animate:!1,pan:!0},!0===t?{animate:!0}:t);var n=this.getSize();this._sizeChanged=!0,this._lastCenter=null;var o=this.getSize(),s=n.divideBy(2).round(),r=o.divideBy(2).round(),a=s.subtract(r);return a.x||a.y?(t.animate&&t.pan?this.panBy(a):(t.pan&&this._rawPanBy(a),this.fire("move"),t.debounceMoveend?(clearTimeout(this._sizeTimer),this._sizeTimer=setTimeout(e(this.fire,this,"moveend"),200)):this.fire("moveend")),this.fire("resize",{oldSize:n,newSize:o})):this},stop:function(){return this.setZoom(this._limitZoom(this._zoom)),this.options.zoomSnap||this.fire("viewreset"),this._stop()},locate:function(t){if(t=this._locateOptions=i({timeout:1e4,watch:!1},t),!("geolocation"in navigator))return this._handleGeolocationError({code:0,message:"Geolocation not supported."}),this;var n=e(this._handleGeolocationResponse,this),o=e(this._handleGeolocationError,this);return t.watch?this._locationWatchId=navigator.geolocation.watchPosition(n,o,t):navigator.geolocation.getCurrentPosition(n,o,t),this},stopLocate:function(){return navigator.geolocation&&navigator.geolocation.clearWatch&&navigator.geolocation.clearWatch(this._locationWatchId),this._locateOptions&&(this._locateOptions.setView=!1),this},_handleGeolocationError:function(t){var i=t.code,e=t.message||(1===i?"permission denied":2===i?"position unavailable":"timeout");this._locateOptions.setView&&!this._loaded&&this.fitWorld(),this.fire("locationerror",{code:i,message:"Geolocation error: "+e+"."})},_handleGeolocationResponse:function(t){var i=new M(t.coords.latitude,t.coords.longitude),e=i.toBounds(t.coords.accuracy),n=this._locateOptions;if(n.setView){var o=this.getBoundsZoom(e);this.setView(i,n.maxZoom?Math.min(o,n.maxZoom):o)}var s={latlng:i,bounds:e,timestamp:t.timestamp};for(var r in t.coords)"number"==typeof t.coords[r]&&(s[r]=t.coords[r]);this.fire("locationfound",s)},addHandler:function(t,i){if(!i)return this;var e=this[t]=new i(this);return this._handlers.push(e),this.options[t]&&e.enable(),this},remove:function(){if(this._initEvents(!0),this._containerId!==this._container._leaflet_id)throw new Error("Map container is being reused by another instance");try{delete this._container._leaflet_id,delete this._containerId}catch(t){this._container._leaflet_id=void 0,this._containerId=void 0}void 0!==this._locationWatchId&&this.stopLocate(),this._stop(),ut(this._mapPane),this._clearControlPos&&this._clearControlPos(),this._clearHandlers(),this._loaded&&this.fire("unload");var t;for(t in this._layers)this._layers[t].remove();for(t in this._panes)ut(this._panes[t]);return this._layers=[],this._panes=[],delete this._mapPane,delete this._renderer,this},createPane:function(t,i){var e=ht("div","leaflet-pane"+(t?" leaflet-"+t.replace("Pane","")+"-pane":""),i||this._mapPane);return t&&(this._panes[t]=e),e},getCenter:function(){return this._checkIfLoaded(),this._lastCenter&&!this._moved()?this._lastCenter:this.layerPointToLatLng(this._getCenterLayerPoint())},getZoom:function(){return this._zoom},getBounds:function(){var t=this.getPixelBounds();return new T(this.unproject(t.getBottomLeft()),this.unproject(t.getTopRight()))},getMinZoom:function(){return void 0===this.options.minZoom?this._layersMinZoom||0:this.options.minZoom},getMaxZoom:function(){return void 0===this.options.maxZoom?void 0===this._layersMaxZoom?1/0:this._layersMaxZoom:this.options.maxZoom},getBoundsZoom:function(t,i,e){t=z(t),e=w(e||[0,0]);var n=this.getZoom()||0,o=this.getMinZoom(),s=this.getMaxZoom(),r=t.getNorthWest(),a=t.getSouthEast(),h=this.getSize().subtract(e),u=b(this.project(a,n),this.project(r,n)).getSize(),l=Ni?this.options.zoomSnap:1,c=h.x/u.x,_=h.y/u.y,d=i?Math.max(c,_):Math.min(c,_);return n=this.getScaleZoom(d,n),l&&(n=Math.round(n/(l/100))*(l/100),n=i?Math.ceil(n/l)*l:Math.floor(n/l)*l),Math.max(o,Math.min(s,n))},getSize:function(){return this._size&&!this._sizeChanged||(this._size=new x(this._container.clientWidth||0,this._container.clientHeight||0),this._sizeChanged=!1),this._size.clone()},getPixelBounds:function(t,i){var e=this._getTopLeftPoint(t,i);return new P(e,e.add(this.getSize()))},getPixelOrigin:function(){return this._checkIfLoaded(),this._pixelOrigin},getPixelWorldBounds:function(t){return this.options.crs.getProjectedBounds(void 0===t?this.getZoom():t)},getPane:function(t){return"string"==typeof t?this._panes[t]:t},getPanes:function(){return this._panes},getContainer:function(){return this._container},getZoomScale:function(t,i){var e=this.options.crs;return i=void 0===i?this._zoom:i,e.scale(t)/e.scale(i)},getScaleZoom:function(t,i){var e=this.options.crs;i=void 0===i?this._zoom:i;var n=e.zoom(t*e.scale(i));return isNaN(n)?1/0:n},project:function(t,i){return i=void 0===i?this._zoom:i,this.options.crs.latLngToPoint(C(t),i)},unproject:function(t,i){return i=void 0===i?this._zoom:i,this.options.crs.pointToLatLng(w(t),i)},layerPointToLatLng:function(t){var i=w(t).add(this.getPixelOrigin());return this.unproject(i)},latLngToLayerPoint:function(t){return this.project(C(t))._round()._subtract(this.getPixelOrigin())},wrapLatLng:function(t){return this.options.crs.wrapLatLng(C(t))},wrapLatLngBounds:function(t){return this.options.crs.wrapLatLngBounds(z(t))},distance:function(t,i){return this.options.crs.distance(C(t),C(i))},containerPointToLayerPoint:function(t){return w(t).subtract(this._getMapPanePos())},layerPointToContainerPoint:function(t){return w(t).add(this._getMapPanePos())},containerPointToLatLng:function(t){var i=this.containerPointToLayerPoint(w(t));return this.layerPointToLatLng(i)},latLngToContainerPoint:function(t){return this.layerPointToContainerPoint(this.latLngToLayerPoint(C(t)))},mouseEventToContainerPoint:function(t){return tt(t,this._container)},mouseEventToLayerPoint:function(t){return this.containerPointToLayerPoint(this.mouseEventToContainerPoint(t))},mouseEventToLatLng:function(t){return this.layerPointToLatLng(this.mouseEventToLayerPoint(t))},_initContainer:function(t){var i=this._container=rt(t);if(!i)throw new Error("Map container not found.");if(i._leaflet_id)throw new Error("Map container is already initialized.");V(i,"scroll",this._onScroll,this),this._containerId=n(i)},_initLayout:function(){var t=this._container;this._fadeAnimated=this.options.fadeAnimation&&Ni,pt(t,"leaflet-container"+(Vi?" leaflet-touch":"")+(Ki?" leaflet-retina":"")+(Li?" leaflet-oldie":"")+(ki?" leaflet-safari":"")+(this._fadeAnimated?" leaflet-fade-anim":""));var i=at(t,"position");"absolute"!==i&&"relative"!==i&&"fixed"!==i&&(t.style.position="relative"),this._initPanes(),this._initControlPos&&this._initControlPos()},_initPanes:function(){var t=this._panes={};this._paneRenderers={},this._mapPane=this.createPane("mapPane",this._container),Lt(this._mapPane,new x(0,0)),this.createPane("tilePane"),this.createPane("shadowPane"),this.createPane("overlayPane"),this.createPane("markerPane"),this.createPane("tooltipPane"),this.createPane("popupPane"),this.options.markerZoomAnimation||(pt(t.markerPane,"leaflet-zoom-hide"),pt(t.shadowPane,"leaflet-zoom-hide"))},_resetView:function(t,i){Lt(this._mapPane,new x(0,0));var e=!this._loaded;this._loaded=!0,i=this._limitZoom(i),this.fire("viewprereset");var n=this._zoom!==i;this._moveStart(n,!1)._move(t,i)._moveEnd(n),this.fire("viewreset"),e&&this.fire("load")},_moveStart:function(t,i){return t&&this.fire("zoomstart"),i||this.fire("movestart"),this},_move:function(t,i,e){void 0===i&&(i=this._zoom);var n=this._zoom!==i;return this._zoom=i,this._lastCenter=t,this._pixelOrigin=this._getNewPixelOrigin(t),(n||e&&e.pinch)&&this.fire("zoom",e),this.fire("move",e)},_moveEnd:function(t){return t&&this.fire("zoomend"),this.fire("moveend")},_stop:function(){return g(this._flyToFrame),this._panAnim&&this._panAnim.stop(),this},_rawPanBy:function(t){Lt(this._mapPane,this._getMapPanePos().subtract(t))},_getZoomSpan:function(){return this.getMaxZoom()-this.getMinZoom()},_panInsideMaxBounds:function(){this._enforcingBounds||this.panInsideBounds(this.options.maxBounds)},_checkIfLoaded:function(){if(!this._loaded)throw new Error("Set map center and zoom first.")},_initEvents:function(t){this._targets={},this._targets[n(this._container)]=this;var i=t?q:V;i(this._container,"click dblclick mousedown mouseup mouseover mouseout mousemove contextmenu keypress",this._handleDOMEvent,this),this.options.trackResize&&i(window,"resize",this._onResize,this),Ni&&this.options.transform3DLimit&&(t?this.off:this.on).call(this,"moveend",this._onMoveEnd)},_onResize:function(){g(this._resizeRequest),this._resizeRequest=f(function(){this.invalidateSize({debounceMoveend:!0})},this)},_onScroll:function(){this._container.scrollTop=0,this._container.scrollLeft=0},_onMoveEnd:function(){var t=this._getMapPanePos();Math.max(Math.abs(t.x),Math.abs(t.y))>=this.options.transform3DLimit&&this._resetView(this.getCenter(),this.getZoom())},_findEventTargets:function(t,i){for(var e,o=[],s="mouseout"===i||"mouseover"===i,r=t.target||t.srcElement,a=!1;r;){if((e=this._targets[n(r)])&&("click"===i||"preclick"===i)&&!t._simulated&&this._draggableMoved(e)){a=!0;break}if(e&&e.listens(i,!0)){if(s&&!ot(r,t))break;if(o.push(e),s)break}if(r===this._container)break;r=r.parentNode}return o.length||a||s||!ot(r,t)||(o=[this]),o},_handleDOMEvent:function(t){if(this._loaded&&!nt(t)){var i=t.type;"mousedown"!==i&&"keypress"!==i||zt(t.target||t.srcElement),this._fireDOMEvent(t,i)}},_mouseEvents:["click","dblclick","mouseover","mouseout","contextmenu"],_fireDOMEvent:function(t,e,n){if("click"===t.type){var o=i({},t);o.type="preclick",this._fireDOMEvent(o,o.type,n)}if(!t._stopped&&(n=(n||[]).concat(this._findEventTargets(t,e))).length){var s=n[0];"contextmenu"===e&&s.listens(e,!0)&&$(t);var r={originalEvent:t};if("keypress"!==t.type){var a=s.getLatLng&&(!s._radius||s._radius<=10);r.containerPoint=a?this.latLngToContainerPoint(s.getLatLng()):this.mouseEventToContainerPoint(t),r.layerPoint=this.containerPointToLayerPoint(r.containerPoint),r.latlng=a?s.getLatLng():this.layerPointToLatLng(r.layerPoint)}for(var h=0;h<n.length;h++)if(n[h].fire(e,r,!0),r.originalEvent._stopped||!1===n[h].options.bubblingMouseEvents&&-1!==d(this._mouseEvents,e))return}},_draggableMoved:function(t){return(t=t.dragging&&t.dragging.enabled()?t:this).dragging&&t.dragging.moved()||this.boxZoom&&this.boxZoom.moved()},_clearHandlers:function(){for(var t=0,i=this._handlers.length;t<i;t++)this._handlers[t].disable()},whenReady:function(t,i){return this._loaded?t.call(i||this,{target:this}):this.on("load",t,i),this},_getMapPanePos:function(){return Pt(this._mapPane)||new x(0,0)},_moved:function(){var t=this._getMapPanePos();return t&&!t.equals([0,0])},_getTopLeftPoint:function(t,i){return(t&&void 0!==i?this._getNewPixelOrigin(t,i):this.getPixelOrigin()).subtract(this._getMapPanePos())},_getNewPixelOrigin:function(t,i){var e=this.getSize()._divideBy(2);return this.project(t,i)._subtract(e)._add(this._getMapPanePos())._round()},_latLngToNewLayerPoint:function(t,i,e){var n=this._getNewPixelOrigin(e,i);return this.project(t,i)._subtract(n)},_latLngBoundsToNewLayerBounds:function(t,i,e){var n=this._getNewPixelOrigin(e,i);return b([this.project(t.getSouthWest(),i)._subtract(n),this.project(t.getNorthWest(),i)._subtract(n),this.project(t.getSouthEast(),i)._subtract(n),this.project(t.getNorthEast(),i)._subtract(n)])},_getCenterLayerPoint:function(){return this.containerPointToLayerPoint(this.getSize()._divideBy(2))},_getCenterOffset:function(t){return this.latLngToLayerPoint(t).subtract(this._getCenterLayerPoint())},_limitCenter:function(t,i,e){if(!e)return t;var n=this.project(t,i),o=this.getSize().divideBy(2),s=new P(n.subtract(o),n.add(o)),r=this._getBoundsOffset(s,e,i);return r.round().equals([0,0])?t:this.unproject(n.add(r),i)},_limitOffset:function(t,i){if(!i)return t;var e=this.getPixelBounds(),n=new P(e.min.add(t),e.max.add(t));return t.add(this._getBoundsOffset(n,i))},_getBoundsOffset:function(t,i,e){var n=b(this.project(i.getNorthEast(),e),this.project(i.getSouthWest(),e)),o=n.min.subtract(t.min),s=n.max.subtract(t.max);return new x(this._rebound(o.x,-s.x),this._rebound(o.y,-s.y))},_rebound:function(t,i){return t+i>0?Math.round(t-i)/2:Math.max(0,Math.ceil(t))-Math.max(0,Math.floor(i))},_limitZoom:function(t){var i=this.getMinZoom(),e=this.getMaxZoom(),n=Ni?this.options.zoomSnap:1;return n&&(t=Math.round(t/n)*n),Math.max(i,Math.min(e,t))},_onPanTransitionStep:function(){this.fire("move")},_onPanTransitionEnd:function(){mt(this._mapPane,"leaflet-pan-anim"),this.fire("moveend")},_tryAnimatedPan:function(t,i){var e=this._getCenterOffset(t)._trunc();return!(!0!==(i&&i.animate)&&!this.getSize().contains(e))&&(this.panBy(e,i),!0)},_createAnimProxy:function(){var t=this._proxy=ht("div","leaflet-proxy leaflet-zoom-animated");this._panes.mapPane.appendChild(t),this.on("zoomanim",function(t){var i=pe,e=this._proxy.style[i];wt(this._proxy,this.project(t.center,t.zoom),this.getZoomScale(t.zoom,1)),e===this._proxy.style[i]&&this._animatingZoom&&this._onZoomTransitionEnd()},this),this.on("load moveend",function(){var t=this.getCenter(),i=this.getZoom();wt(this._proxy,this.project(t,i),this.getZoomScale(i,1))},this),this._on("unload",this._destroyAnimProxy,this)},_destroyAnimProxy:function(){ut(this._proxy),delete this._proxy},_catchTransitionEnd:function(t){this._animatingZoom&&t.propertyName.indexOf("transform")>=0&&this._onZoomTransitionEnd()},_nothingToAnimate:function(){return!this._container.getElementsByClassName("leaflet-zoom-animated").length},_tryAnimatedZoom:function(t,i,e){if(this._animatingZoom)return!0;if(e=e||{},!this._zoomAnimated||!1===e.animate||this._nothingToAnimate()||Math.abs(i-this._zoom)>this.options.zoomAnimationThreshold)return!1;var n=this.getZoomScale(i),o=this._getCenterOffset(t)._divideBy(1-1/n);return!(!0!==e.animate&&!this.getSize().contains(o))&&(f(function(){this._moveStart(!0,!1)._animateZoom(t,i,!0)},this),!0)},_animateZoom:function(t,i,n,o){this._mapPane&&(n&&(this._animatingZoom=!0,this._animateToCenter=t,this._animateToZoom=i,pt(this._mapPane,"leaflet-zoom-anim")),this.fire("zoomanim",{center:t,zoom:i,noUpdate:o}),setTimeout(e(this._onZoomTransitionEnd,this),250))},_onZoomTransitionEnd:function(){this._animatingZoom&&(this._mapPane&&mt(this._mapPane,"leaflet-zoom-anim"),this._animatingZoom=!1,this._move(this._animateToCenter,this._animateToZoom),f(function(){this._moveEnd(!0)},this))}}),Pe=v.extend({options:{position:"topright"},initialize:function(t){l(this,t)},getPosition:function(){return this.options.position},setPosition:function(t){var i=this._map;return i&&i.removeControl(this),this.options.position=t,i&&i.addControl(this),this},getContainer:function(){return this._container},addTo:function(t){this.remove(),this._map=t;var i=this._container=this.onAdd(t),e=this.getPosition(),n=t._controlCorners[e];return pt(i,"leaflet-control"),-1!==e.indexOf("bottom")?n.insertBefore(i,n.firstChild):n.appendChild(i),this},remove:function(){return this._map?(ut(this._container),this.onRemove&&this.onRemove(this._map),this._map=null,this):this},_refocusOnMap:function(t){this._map&&t&&t.screenX>0&&t.screenY>0&&this._map.getContainer().focus()}}),be=function(t){return new Pe(t)};Le.include({addControl:function(t){return t.addTo(this),this},removeControl:function(t){return t.remove(),this},_initControlPos:function(){function t(t,o){var s=e+t+" "+e+o;i[t+o]=ht("div",s,n)}var i=this._controlCorners={},e="leaflet-",n=this._controlContainer=ht("div",e+"control-container",this._container);t("top","left"),t("top","right"),t("bottom","left"),t("bottom","right")},_clearControlPos:function(){for(var t in this._controlCorners)ut(this._controlCorners[t]);ut(this._controlContainer),delete this._controlCorners,delete this._controlContainer}});var Te=Pe.extend({options:{collapsed:!0,position:"topright",autoZIndex:!0,hideSingleBase:!1,sortLayers:!1,sortFunction:function(t,i,e,n){return e<n?-1:n<e?1:0}},initialize:function(t,i,e){l(this,e),this._layerControlInputs=[],this._layers=[],this._lastZIndex=0,this._handlingClick=!1;for(var n in t)this._addLayer(t[n],n);for(n in i)this._addLayer(i[n],n,!0)},onAdd:function(t){this._initLayout(),this._update(),this._map=t,t.on("zoomend",this._checkDisabledLayers,this);for(var i=0;i<this._layers.length;i++)this._layers[i].layer.on("add remove",this._onLayerChange,this);return this._container},addTo:function(t){return Pe.prototype.addTo.call(this,t),this._expandIfNotCollapsed()},onRemove:function(){this._map.off("zoomend",this._checkDisabledLayers,this);for(var t=0;t<this._layers.length;t++)this._layers[t].layer.off("add remove",this._onLayerChange,this)},addBaseLayer:function(t,i){return this._addLayer(t,i),this._map?this._update():this},addOverlay:function(t,i){return this._addLayer(t,i,!0),this._map?this._update():this},removeLayer:function(t){t.off("add remove",this._onLayerChange,this);var i=this._getLayer(n(t));return i&&this._layers.splice(this._layers.indexOf(i),1),this._map?this._update():this},expand:function(){pt(this._container,"leaflet-control-layers-expanded"),this._form.style.height=null;var t=this._map.getSize().y-(this._container.offsetTop+50);return t<this._form.clientHeight?(pt(this._form,"leaflet-control-layers-scrollbar"),this._form.style.height=t+"px"):mt(this._form,"leaflet-control-layers-scrollbar"),this._checkDisabledLayers(),this},collapse:function(){return mt(this._container,"leaflet-control-layers-expanded"),this},_initLayout:function(){var t="leaflet-control-layers",i=this._container=ht("div",t),e=this.options.collapsed;i.setAttribute("aria-haspopup",!0),J(i),X(i);var n=this._form=ht("form",t+"-list");e&&(this._map.on("click",this.collapse,this),Ti||V(i,{mouseenter:this.expand,mouseleave:this.collapse},this));var o=this._layersLink=ht("a",t+"-toggle",i);o.href="#",o.title="Layers",Vi?(V(o,"click",Q),V(o,"click",this.expand,this)):V(o,"focus",this.expand,this),e||this.expand(),this._baseLayersList=ht("div",t+"-base",n),this._separator=ht("div",t+"-separator",n),this._overlaysList=ht("div",t+"-overlays",n),i.appendChild(n)},_getLayer:function(t){for(var i=0;i<this._layers.length;i++)if(this._layers[i]&&n(this._layers[i].layer)===t)return this._layers[i]},_addLayer:function(t,i,n){this._map&&t.on("add remove",this._onLayerChange,this),this._layers.push({layer:t,name:i,overlay:n}),this.options.sortLayers&&this._layers.sort(e(function(t,i){return this.options.sortFunction(t.layer,i.layer,t.name,i.name)},this)),this.options.autoZIndex&&t.setZIndex&&(this._lastZIndex++,t.setZIndex(this._lastZIndex)),this._expandIfNotCollapsed()},_update:function(){if(!this._container)return this;lt(this._baseLayersList),lt(this._overlaysList),this._layerControlInputs=[];var t,i,e,n,o=0;for(e=0;e<this._layers.length;e++)n=this._layers[e],this._addItem(n),i=i||n.overlay,t=t||!n.overlay,o+=n.overlay?0:1;return this.options.hideSingleBase&&(t=t&&o>1,this._baseLayersList.style.display=t?"":"none"),this._separator.style.display=i&&t?"":"none",this},_onLayerChange:function(t){this._handlingClick||this._update();var i=this._getLayer(n(t.target)),e=i.overlay?"add"===t.type?"overlayadd":"overlayremove":"add"===t.type?"baselayerchange":null;e&&this._map.fire(e,i)},_createRadioElement:function(t,i){var e='<input type="radio" class="leaflet-control-layers-selector" name="'+t+'"'+(i?' checked="checked"':"")+"/>",n=document.createElement("div");return n.innerHTML=e,n.firstChild},_addItem:function(t){var i,e=document.createElement("label"),o=this._map.hasLayer(t.layer);t.overlay?((i=document.createElement("input")).type="checkbox",i.className="leaflet-control-layers-selector",i.defaultChecked=o):i=this._createRadioElement("leaflet-base-layers",o),this._layerControlInputs.push(i),i.layerId=n(t.layer),V(i,"click",this._onInputClick,this);var s=document.createElement("span");s.innerHTML=" "+t.name;var r=document.createElement("div");return e.appendChild(r),r.appendChild(i),r.appendChild(s),(t.overlay?this._overlaysList:this._baseLayersList).appendChild(e),this._checkDisabledLayers(),e},_onInputClick:function(){var t,i,e=this._layerControlInputs,n=[],o=[];this._handlingClick=!0;for(var s=e.length-1;s>=0;s--)t=e[s],i=this._getLayer(t.layerId).layer,t.checked?n.push(i):t.checked||o.push(i);for(s=0;s<o.length;s++)this._map.hasLayer(o[s])&&this._map.removeLayer(o[s]);for(s=0;s<n.length;s++)this._map.hasLayer(n[s])||this._map.addLayer(n[s]);this._handlingClick=!1,this._refocusOnMap()},_checkDisabledLayers:function(){for(var t,i,e=this._layerControlInputs,n=this._map.getZoom(),o=e.length-1;o>=0;o--)t=e[o],i=this._getLayer(t.layerId).layer,t.disabled=void 0!==i.options.minZoom&&n<i.options.minZoom||void 0!==i.options.maxZoom&&n>i.options.maxZoom},_expandIfNotCollapsed:function(){return this._map&&!this.options.collapsed&&this.expand(),this},_expand:function(){return this.expand()},_collapse:function(){return this.collapse()}}),ze=Pe.extend({options:{position:"topleft",zoomInText:"+",zoomInTitle:"Zoom in",zoomOutText:"&#x2212;",zoomOutTitle:"Zoom out"},onAdd:function(t){var i="leaflet-control-zoom",e=ht("div",i+" leaflet-bar"),n=this.options;return this._zoomInButton=this._createButton(n.zoomInText,n.zoomInTitle,i+"-in",e,this._zoomIn),this._zoomOutButton=this._createButton(n.zoomOutText,n.zoomOutTitle,i+"-out",e,this._zoomOut),this._updateDisabled(),t.on("zoomend zoomlevelschange",this._updateDisabled,this),e},onRemove:function(t){t.off("zoomend zoomlevelschange",this._updateDisabled,this)},disable:function(){return this._disabled=!0,this._updateDisabled(),this},enable:function(){return this._disabled=!1,this._updateDisabled(),this},_zoomIn:function(t){!this._disabled&&this._map._zoom<this._map.getMaxZoom()&&this._map.zoomIn(this._map.options.zoomDelta*(t.shiftKey?3:1))},_zoomOut:function(t){!this._disabled&&this._map._zoom>this._map.getMinZoom()&&this._map.zoomOut(this._map.options.zoomDelta*(t.shiftKey?3:1))},_createButton:function(t,i,e,n,o){var s=ht("a",e,n);return s.innerHTML=t,s.href="#",s.title=i,s.setAttribute("role","button"),s.setAttribute("aria-label",i),J(s),V(s,"click",Q),V(s,"click",o,this),V(s,"click",this._refocusOnMap,this),s},_updateDisabled:function(){var t=this._map,i="leaflet-disabled";mt(this._zoomInButton,i),mt(this._zoomOutButton,i),(this._disabled||t._zoom===t.getMinZoom())&&pt(this._zoomOutButton,i),(this._disabled||t._zoom===t.getMaxZoom())&&pt(this._zoomInButton,i)}});Le.mergeOptions({zoomControl:!0}),Le.addInitHook(function(){this.options.zoomControl&&(this.zoomControl=new ze,this.addControl(this.zoomControl))});var Me=Pe.extend({options:{position:"bottomleft",maxWidth:100,metric:!0,imperial:!0},onAdd:function(t){var i=ht("div","leaflet-control-scale"),e=this.options;return this._addScales(e,"leaflet-control-scale-line",i),t.on(e.updateWhenIdle?"moveend":"move",this._update,this),t.whenReady(this._update,this),i},onRemove:function(t){t.off(this.options.updateWhenIdle?"moveend":"move",this._update,this)},_addScales:function(t,i,e){t.metric&&(this._mScale=ht("div",i,e)),t.imperial&&(this._iScale=ht("div",i,e))},_update:function(){var t=this._map,i=t.getSize().y/2,e=t.distance(t.containerPointToLatLng([0,i]),t.containerPointToLatLng([this.options.maxWidth,i]));this._updateScales(e)},_updateScales:function(t){this.options.metric&&t&&this._updateMetric(t),this.options.imperial&&t&&this._updateImperial(t)},_updateMetric:function(t){var i=this._getRoundNum(t),e=i<1e3?i+" m":i/1e3+" km";this._updateScale(this._mScale,e,i/t)},_updateImperial:function(t){var i,e,n,o=3.2808399*t;o>5280?(i=o/5280,e=this._getRoundNum(i),this._updateScale(this._iScale,e+" mi",e/i)):(n=this._getRoundNum(o),this._updateScale(this._iScale,n+" ft",n/o))},_updateScale:function(t,i,e){t.style.width=Math.round(this.options.maxWidth*e)+"px",t.innerHTML=i},_getRoundNum:function(t){var i=Math.pow(10,(Math.floor(t)+"").length-1),e=t/i;return e=e>=10?10:e>=5?5:e>=3?3:e>=2?2:1,i*e}}),Ce=Pe.extend({options:{position:"bottomright",prefix:'<a href="https://leafletjs.com" title="A JS library for interactive maps">Leaflet</a>'},initialize:function(t){l(this,t),this._attributions={}},onAdd:function(t){t.attributionControl=this,this._container=ht("div","leaflet-control-attribution"),J(this._container);for(var i in t._layers)t._layers[i].getAttribution&&this.addAttribution(t._layers[i].getAttribution());return this._update(),this._container},setPrefix:function(t){return this.options.prefix=t,this._update(),this},addAttribution:function(t){return t?(this._attributions[t]||(this._attributions[t]=0),this._attributions[t]++,this._update(),this):this},removeAttribution:function(t){return t?(this._attributions[t]&&(this._attributions[t]--,this._update()),this):this},_update:function(){if(this._map){var t=[];for(var i in this._attributions)this._attributions[i]&&t.push(i);var e=[];this.options.prefix&&e.push(this.options.prefix),t.length&&e.push(t.join(", ")),this._container.innerHTML=e.join(" | ")}}});Le.mergeOptions({attributionControl:!0}),Le.addInitHook(function(){this.options.attributionControl&&(new Ce).addTo(this)});Pe.Layers=Te,Pe.Zoom=ze,Pe.Scale=Me,Pe.Attribution=Ce,be.layers=function(t,i,e){return new Te(t,i,e)},be.zoom=function(t){return new ze(t)},be.scale=function(t){return new Me(t)},be.attribution=function(t){return new Ce(t)};var Ze=v.extend({initialize:function(t){this._map=t},enable:function(){return this._enabled?this:(this._enabled=!0,this.addHooks(),this)},disable:function(){return this._enabled?(this._enabled=!1,this.removeHooks(),this):this},enabled:function(){return!!this._enabled}});Ze.addTo=function(t,i){return t.addHandler(i,this),this};var Se,Ee={Events:hi},ke=Vi?"touchstart mousedown":"mousedown",Ae={mousedown:"mouseup",touchstart:"touchend",pointerdown:"touchend",MSPointerDown:"touchend"},Ie={mousedown:"mousemove",touchstart:"touchmove",pointerdown:"touchmove",MSPointerDown:"touchmove"},Be=ui.extend({options:{clickTolerance:3},initialize:function(t,i,e,n){l(this,n),this._element=t,this._dragStartTarget=i||t,this._preventOutline=e},enable:function(){this._enabled||(V(this._dragStartTarget,ke,this._onDown,this),this._enabled=!0)},disable:function(){this._enabled&&(Be._dragging===this&&this.finishDrag(),q(this._dragStartTarget,ke,this._onDown,this),this._enabled=!1,this._moved=!1)},_onDown:function(t){if(!t._simulated&&this._enabled&&(this._moved=!1,!dt(this._element,"leaflet-zoom-anim")&&!(Be._dragging||t.shiftKey||1!==t.which&&1!==t.button&&!t.touches||(Be._dragging=this,this._preventOutline&&zt(this._element),bt(),mi(),this._moving)))){this.fire("down");var i=t.touches?t.touches[0]:t;this._startPoint=new x(i.clientX,i.clientY),V(document,Ie[t.type],this._onMove,this),V(document,Ae[t.type],this._onUp,this)}},_onMove:function(t){if(!t._simulated&&this._enabled)if(t.touches&&t.touches.length>1)this._moved=!0;else{var i=t.touches&&1===t.touches.length?t.touches[0]:t,e=new x(i.clientX,i.clientY).subtract(this._startPoint);(e.x||e.y)&&(Math.abs(e.x)+Math.abs(e.y)<this.options.clickTolerance||($(t),this._moved||(this.fire("dragstart"),this._moved=!0,this._startPos=Pt(this._element).subtract(e),pt(document.body,"leaflet-dragging"),this._lastTarget=t.target||t.srcElement,window.SVGElementInstance&&this._lastTarget instanceof SVGElementInstance&&(this._lastTarget=this._lastTarget.correspondingUseElement),pt(this._lastTarget,"leaflet-drag-target")),this._newPos=this._startPos.add(e),this._moving=!0,g(this._animRequest),this._lastEvent=t,this._animRequest=f(this._updatePosition,this,!0)))}},_updatePosition:function(){var t={originalEvent:this._lastEvent};this.fire("predrag",t),Lt(this._element,this._newPos),this.fire("drag",t)},_onUp:function(t){!t._simulated&&this._enabled&&this.finishDrag()},finishDrag:function(){mt(document.body,"leaflet-dragging"),this._lastTarget&&(mt(this._lastTarget,"leaflet-drag-target"),this._lastTarget=null);for(var t in Ie)q(document,Ie[t],this._onMove,this),q(document,Ae[t],this._onUp,this);Tt(),fi(),this._moved&&this._moving&&(g(this._animRequest),this.fire("dragend",{distance:this._newPos.distanceTo(this._startPos)})),this._moving=!1,Be._dragging=!1}}),Oe=(Object.freeze||Object)({simplify:Ct,pointToSegmentDistance:Zt,closestPointOnSegment:function(t,i,e){return Rt(t,i,e)},clipSegment:At,_getEdgeIntersection:It,_getBitCode:Bt,_sqClosestPointOnSegment:Rt,isFlat:Dt,_flat:Nt}),Re=(Object.freeze||Object)({clipPolygon:jt}),De={project:function(t){return new x(t.lng,t.lat)},unproject:function(t){return new M(t.y,t.x)},bounds:new P([-180,-90],[180,90])},Ne={R:6378137,R_MINOR:6356752.314245179,bounds:new P([-20037508.34279,-15496570.73972],[20037508.34279,18764656.23138]),project:function(t){var i=Math.PI/180,e=this.R,n=t.lat*i,o=this.R_MINOR/e,s=Math.sqrt(1-o*o),r=s*Math.sin(n),a=Math.tan(Math.PI/4-n/2)/Math.pow((1-r)/(1+r),s/2);return n=-e*Math.log(Math.max(a,1e-10)),new x(t.lng*i*e,n)},unproject:function(t){for(var i,e=180/Math.PI,n=this.R,o=this.R_MINOR/n,s=Math.sqrt(1-o*o),r=Math.exp(-t.y/n),a=Math.PI/2-2*Math.atan(r),h=0,u=.1;h<15&&Math.abs(u)>1e-7;h++)i=s*Math.sin(a),i=Math.pow((1-i)/(1+i),s/2),a+=u=Math.PI/2-2*Math.atan(r*i)-a;return new M(a*e,t.x*e/n)}},je=(Object.freeze||Object)({LonLat:De,Mercator:Ne,SphericalMercator:di}),We=i({},_i,{code:"EPSG:3395",projection:Ne,transformation:function(){var t=.5/(Math.PI*Ne.R);return S(t,.5,-t,.5)}()}),He=i({},_i,{code:"EPSG:4326",projection:De,transformation:S(1/180,1,-1/180,.5)}),Fe=i({},ci,{projection:De,transformation:S(1,0,-1,0),scale:function(t){return Math.pow(2,t)},zoom:function(t){return Math.log(t)/Math.LN2},distance:function(t,i){var e=i.lng-t.lng,n=i.lat-t.lat;return Math.sqrt(e*e+n*n)},infinite:!0});ci.Earth=_i,ci.EPSG3395=We,ci.EPSG3857=vi,ci.EPSG900913=yi,ci.EPSG4326=He,ci.Simple=Fe;var Ue=ui.extend({options:{pane:"overlayPane",attribution:null,bubblingMouseEvents:!0},addTo:function(t){return t.addLayer(this),this},remove:function(){return this.removeFrom(this._map||this._mapToAdd)},removeFrom:function(t){return t&&t.removeLayer(this),this},getPane:function(t){return this._map.getPane(t?this.options[t]||t:this.options.pane)},addInteractiveTarget:function(t){return this._map._targets[n(t)]=this,this},removeInteractiveTarget:function(t){return delete this._map._targets[n(t)],this},getAttribution:function(){return this.options.attribution},_layerAdd:function(t){var i=t.target;if(i.hasLayer(this)){if(this._map=i,this._zoomAnimated=i._zoomAnimated,this.getEvents){var e=this.getEvents();i.on(e,this),this.once("remove",function(){i.off(e,this)},this)}this.onAdd(i),this.getAttribution&&i.attributionControl&&i.attributionControl.addAttribution(this.getAttribution()),this.fire("add"),i.fire("layeradd",{layer:this})}}});Le.include({addLayer:function(t){if(!t._layerAdd)throw new Error("The provided object is not a Layer.");var i=n(t);return this._layers[i]?this:(this._layers[i]=t,t._mapToAdd=this,t.beforeAdd&&t.beforeAdd(this),this.whenReady(t._layerAdd,t),this)},removeLayer:function(t){var i=n(t);return this._layers[i]?(this._loaded&&t.onRemove(this),t.getAttribution&&this.attributionControl&&this.attributionControl.removeAttribution(t.getAttribution()),delete this._layers[i],this._loaded&&(this.fire("layerremove",{layer:t}),t.fire("remove")),t._map=t._mapToAdd=null,this):this},hasLayer:function(t){return!!t&&n(t)in this._layers},eachLayer:function(t,i){for(var e in this._layers)t.call(i,this._layers[e]);return this},_addLayers:function(t){for(var i=0,e=(t=t?ei(t)?t:[t]:[]).length;i<e;i++)this.addLayer(t[i])},_addZoomLimit:function(t){!isNaN(t.options.maxZoom)&&isNaN(t.options.minZoom)||(this._zoomBoundLayers[n(t)]=t,this._updateZoomLevels())},_removeZoomLimit:function(t){var i=n(t);this._zoomBoundLayers[i]&&(delete this._zoomBoundLayers[i],this._updateZoomLevels())},_updateZoomLevels:function(){var t=1/0,i=-1/0,e=this._getZoomSpan();for(var n in this._zoomBoundLayers){var o=this._zoomBoundLayers[n].options;t=void 0===o.minZoom?t:Math.min(t,o.minZoom),i=void 0===o.maxZoom?i:Math.max(i,o.maxZoom)}this._layersMaxZoom=i===-1/0?void 0:i,this._layersMinZoom=t===1/0?void 0:t,e!==this._getZoomSpan()&&this.fire("zoomlevelschange"),void 0===this.options.maxZoom&&this._layersMaxZoom&&this.getZoom()>this._layersMaxZoom&&this.setZoom(this._layersMaxZoom),void 0===this.options.minZoom&&this._layersMinZoom&&this.getZoom()<this._layersMinZoom&&this.setZoom(this._layersMinZoom)}});var Ve=Ue.extend({initialize:function(t,i){l(this,i),this._layers={};var e,n;if(t)for(e=0,n=t.length;e<n;e++)this.addLayer(t[e])},addLayer:function(t){var i=this.getLayerId(t);return this._layers[i]=t,this._map&&this._map.addLayer(t),this},removeLayer:function(t){var i=t in this._layers?t:this.getLayerId(t);return this._map&&this._layers[i]&&this._map.removeLayer(this._layers[i]),delete this._layers[i],this},hasLayer:function(t){return!!t&&(t in this._layers||this.getLayerId(t)in this._layers)},clearLayers:function(){return this.eachLayer(this.removeLayer,this)},invoke:function(t){var i,e,n=Array.prototype.slice.call(arguments,1);for(i in this._layers)(e=this._layers[i])[t]&&e[t].apply(e,n);return this},onAdd:function(t){this.eachLayer(t.addLayer,t)},onRemove:function(t){this.eachLayer(t.removeLayer,t)},eachLayer:function(t,i){for(var e in this._layers)t.call(i,this._layers[e]);return this},getLayer:function(t){return this._layers[t]},getLayers:function(){var t=[];return this.eachLayer(t.push,t),t},setZIndex:function(t){return this.invoke("setZIndex",t)},getLayerId:function(t){return n(t)}}),qe=Ve.extend({addLayer:function(t){return this.hasLayer(t)?this:(t.addEventParent(this),Ve.prototype.addLayer.call(this,t),this.fire("layeradd",{layer:t}))},removeLayer:function(t){return this.hasLayer(t)?(t in this._layers&&(t=this._layers[t]),t.removeEventParent(this),Ve.prototype.removeLayer.call(this,t),this.fire("layerremove",{layer:t})):this},setStyle:function(t){return this.invoke("setStyle",t)},bringToFront:function(){return this.invoke("bringToFront")},bringToBack:function(){return this.invoke("bringToBack")},getBounds:function(){var t=new T;for(var i in this._layers){var e=this._layers[i];t.extend(e.getBounds?e.getBounds():e.getLatLng())}return t}}),Ge=v.extend({options:{popupAnchor:[0,0],tooltipAnchor:[0,0]},initialize:function(t){l(this,t)},createIcon:function(t){return this._createIcon("icon",t)},createShadow:function(t){return this._createIcon("shadow",t)},_createIcon:function(t,i){var e=this._getIconUrl(t);if(!e){if("icon"===t)throw new Error("iconUrl not set in Icon options (see the docs).");return null}var n=this._createImg(e,i&&"IMG"===i.tagName?i:null);return this._setIconStyles(n,t),n},_setIconStyles:function(t,i){var e=this.options,n=e[i+"Size"];"number"==typeof n&&(n=[n,n]);var o=w(n),s=w("shadow"===i&&e.shadowAnchor||e.iconAnchor||o&&o.divideBy(2,!0));t.className="leaflet-marker-"+i+" "+(e.className||""),s&&(t.style.marginLeft=-s.x+"px",t.style.marginTop=-s.y+"px"),o&&(t.style.width=o.x+"px",t.style.height=o.y+"px")},_createImg:function(t,i){return i=i||document.createElement("img"),i.src=t,i},_getIconUrl:function(t){return Ki&&this.options[t+"RetinaUrl"]||this.options[t+"Url"]}}),Ke=Ge.extend({options:{iconUrl:"marker-icon.png",iconRetinaUrl:"marker-icon-2x.png",shadowUrl:"marker-shadow.png",iconSize:[25,41],iconAnchor:[12,41],popupAnchor:[1,-34],tooltipAnchor:[16,-28],shadowSize:[41,41]},_getIconUrl:function(t){return Ke.imagePath||(Ke.imagePath=this._detectIconPath()),(this.options.imagePath||Ke.imagePath)+Ge.prototype._getIconUrl.call(this,t)},_detectIconPath:function(){var t=ht("div","leaflet-default-icon-path",document.body),i=at(t,"background-image")||at(t,"backgroundImage");return document.body.removeChild(t),i=null===i||0!==i.indexOf("url")?"":i.replace(/^url\(["']?/,"").replace(/marker-icon\.png["']?\)$/,"")}}),Ye=Ze.extend({initialize:function(t){this._marker=t},addHooks:function(){var t=this._marker._icon;this._draggable||(this._draggable=new Be(t,t,!0)),this._draggable.on({dragstart:this._onDragStart,predrag:this._onPreDrag,drag:this._onDrag,dragend:this._onDragEnd},this).enable(),pt(t,"leaflet-marker-draggable")},removeHooks:function(){this._draggable.off({dragstart:this._onDragStart,predrag:this._onPreDrag,drag:this._onDrag,dragend:this._onDragEnd},this).disable(),this._marker._icon&&mt(this._marker._icon,"leaflet-marker-draggable")},moved:function(){return this._draggable&&this._draggable._moved},_adjustPan:function(t){var i=this._marker,e=i._map,n=this._marker.options.autoPanSpeed,o=this._marker.options.autoPanPadding,s=L.DomUtil.getPosition(i._icon),r=e.getPixelBounds(),a=e.getPixelOrigin(),h=b(r.min._subtract(a).add(o),r.max._subtract(a).subtract(o));if(!h.contains(s)){var u=w((Math.max(h.max.x,s.x)-h.max.x)/(r.max.x-h.max.x)-(Math.min(h.min.x,s.x)-h.min.x)/(r.min.x-h.min.x),(Math.max(h.max.y,s.y)-h.max.y)/(r.max.y-h.max.y)-(Math.min(h.min.y,s.y)-h.min.y)/(r.min.y-h.min.y)).multiplyBy(n);e.panBy(u,{animate:!1}),this._draggable._newPos._add(u),this._draggable._startPos._add(u),L.DomUtil.setPosition(i._icon,this._draggable._newPos),this._onDrag(t),this._panRequest=f(this._adjustPan.bind(this,t))}},_onDragStart:function(){this._oldLatLng=this._marker.getLatLng(),this._marker.closePopup().fire("movestart").fire("dragstart")},_onPreDrag:function(t){this._marker.options.autoPan&&(g(this._panRequest),this._panRequest=f(this._adjustPan.bind(this,t)))},_onDrag:function(t){var i=this._marker,e=i._shadow,n=Pt(i._icon),o=i._map.layerPointToLatLng(n);e&&Lt(e,n),i._latlng=o,t.latlng=o,t.oldLatLng=this._oldLatLng,i.fire("move",t).fire("drag",t)},_onDragEnd:function(t){g(this._panRequest),delete this._oldLatLng,this._marker.fire("moveend").fire("dragend",t)}}),Xe=Ue.extend({options:{icon:new Ke,interactive:!0,draggable:!1,autoPan:!1,autoPanPadding:[50,50],autoPanSpeed:10,keyboard:!0,title:"",alt:"",zIndexOffset:0,opacity:1,riseOnHover:!1,riseOffset:250,pane:"markerPane",bubblingMouseEvents:!1},initialize:function(t,i){l(this,i),this._latlng=C(t)},onAdd:function(t){this._zoomAnimated=this._zoomAnimated&&t.options.markerZoomAnimation,this._zoomAnimated&&t.on("zoomanim",this._animateZoom,this),this._initIcon(),this.update()},onRemove:function(t){this.dragging&&this.dragging.enabled()&&(this.options.draggable=!0,this.dragging.removeHooks()),delete this.dragging,this._zoomAnimated&&t.off("zoomanim",this._animateZoom,this),this._removeIcon(),this._removeShadow()},getEvents:function(){return{zoom:this.update,viewreset:this.update}},getLatLng:function(){return this._latlng},setLatLng:function(t){var i=this._latlng;return this._latlng=C(t),this.update(),this.fire("move",{oldLatLng:i,latlng:this._latlng})},setZIndexOffset:function(t){return this.options.zIndexOffset=t,this.update()},setIcon:function(t){return this.options.icon=t,this._map&&(this._initIcon(),this.update()),this._popup&&this.bindPopup(this._popup,this._popup.options),this},getElement:function(){return this._icon},update:function(){if(this._icon&&this._map){var t=this._map.latLngToLayerPoint(this._latlng).round();this._setPos(t)}return this},_initIcon:function(){var t=this.options,i="leaflet-zoom-"+(this._zoomAnimated?"animated":"hide"),e=t.icon.createIcon(this._icon),n=!1;e!==this._icon&&(this._icon&&this._removeIcon(),n=!0,t.title&&(e.title=t.title),"IMG"===e.tagName&&(e.alt=t.alt||"")),pt(e,i),t.keyboard&&(e.tabIndex="0"),this._icon=e,t.riseOnHover&&this.on({mouseover:this._bringToFront,mouseout:this._resetZIndex});var o=t.icon.createShadow(this._shadow),s=!1;o!==this._shadow&&(this._removeShadow(),s=!0),o&&(pt(o,i),o.alt=""),this._shadow=o,t.opacity<1&&this._updateOpacity(),n&&this.getPane().appendChild(this._icon),this._initInteraction(),o&&s&&this.getPane("shadowPane").appendChild(this._shadow)},_removeIcon:function(){this.options.riseOnHover&&this.off({mouseover:this._bringToFront,mouseout:this._resetZIndex}),ut(this._icon),this.removeInteractiveTarget(this._icon),this._icon=null},_removeShadow:function(){this._shadow&&ut(this._shadow),this._shadow=null},_setPos:function(t){Lt(this._icon,t),this._shadow&&Lt(this._shadow,t),this._zIndex=t.y+this.options.zIndexOffset,this._resetZIndex()},_updateZIndex:function(t){this._icon.style.zIndex=this._zIndex+t},_animateZoom:function(t){var i=this._map._latLngToNewLayerPoint(this._latlng,t.zoom,t.center).round();this._setPos(i)},_initInteraction:function(){if(this.options.interactive&&(pt(this._icon,"leaflet-interactive"),this.addInteractiveTarget(this._icon),Ye)){var t=this.options.draggable;this.dragging&&(t=this.dragging.enabled(),this.dragging.disable()),this.dragging=new Ye(this),t&&this.dragging.enable()}},setOpacity:function(t){return this.options.opacity=t,this._map&&this._updateOpacity(),this},_updateOpacity:function(){var t=this.options.opacity;vt(this._icon,t),this._shadow&&vt(this._shadow,t)},_bringToFront:function(){this._updateZIndex(this.options.riseOffset)},_resetZIndex:function(){this._updateZIndex(0)},_getPopupAnchor:function(){return this.options.icon.options.popupAnchor},_getTooltipAnchor:function(){return this.options.icon.options.tooltipAnchor}}),Je=Ue.extend({options:{stroke:!0,color:"#3388ff",weight:3,opacity:1,lineCap:"round",lineJoin:"round",dashArray:null,dashOffset:null,fill:!1,fillColor:null,fillOpacity:.2,fillRule:"evenodd",interactive:!0,bubblingMouseEvents:!0},beforeAdd:function(t){this._renderer=t.getRenderer(this)},onAdd:function(){this._renderer._initPath(this),this._reset(),this._renderer._addPath(this)},onRemove:function(){this._renderer._removePath(this)},redraw:function(){return this._map&&this._renderer._updatePath(this),this},setStyle:function(t){return l(this,t),this._renderer&&this._renderer._updateStyle(this),this},bringToFront:function(){return this._renderer&&this._renderer._bringToFront(this),this},bringToBack:function(){return this._renderer&&this._renderer._bringToBack(this),this},getElement:function(){return this._path},_reset:function(){this._project(),this._update()},_clickTolerance:function(){return(this.options.stroke?this.options.weight/2:0)+this._renderer.options.tolerance}}),$e=Je.extend({options:{fill:!0,radius:10},initialize:function(t,i){l(this,i),this._latlng=C(t),this._radius=this.options.radius},setLatLng:function(t){return this._latlng=C(t),this.redraw(),this.fire("move",{latlng:this._latlng})},getLatLng:function(){return this._latlng},setRadius:function(t){return this.options.radius=this._radius=t,this.redraw()},getRadius:function(){return this._radius},setStyle:function(t){var i=t&&t.radius||this._radius;return Je.prototype.setStyle.call(this,t),this.setRadius(i),this},_project:function(){this._point=this._map.latLngToLayerPoint(this._latlng),this._updateBounds()},_updateBounds:function(){var t=this._radius,i=this._radiusY||t,e=this._clickTolerance(),n=[t+e,i+e];this._pxBounds=new P(this._point.subtract(n),this._point.add(n))},_update:function(){this._map&&this._updatePath()},_updatePath:function(){this._renderer._updateCircle(this)},_empty:function(){return this._radius&&!this._renderer._bounds.intersects(this._pxBounds)},_containsPoint:function(t){return t.distanceTo(this._point)<=this._radius+this._clickTolerance()}}),Qe=$e.extend({initialize:function(t,e,n){if("number"==typeof e&&(e=i({},n,{radius:e})),l(this,e),this._latlng=C(t),isNaN(this.options.radius))throw new Error("Circle radius cannot be NaN");this._mRadius=this.options.radius},setRadius:function(t){return this._mRadius=t,this.redraw()},getRadius:function(){return this._mRadius},getBounds:function(){var t=[this._radius,this._radiusY||this._radius];return new T(this._map.layerPointToLatLng(this._point.subtract(t)),this._map.layerPointToLatLng(this._point.add(t)))},setStyle:Je.prototype.setStyle,_project:function(){var t=this._latlng.lng,i=this._latlng.lat,e=this._map,n=e.options.crs;if(n.distance===_i.distance){var o=Math.PI/180,s=this._mRadius/_i.R/o,r=e.project([i+s,t]),a=e.project([i-s,t]),h=r.add(a).divideBy(2),u=e.unproject(h).lat,l=Math.acos((Math.cos(s*o)-Math.sin(i*o)*Math.sin(u*o))/(Math.cos(i*o)*Math.cos(u*o)))/o;(isNaN(l)||0===l)&&(l=s/Math.cos(Math.PI/180*i)),this._point=h.subtract(e.getPixelOrigin()),this._radius=isNaN(l)?0:h.x-e.project([u,t-l]).x,this._radiusY=h.y-r.y}else{var c=n.unproject(n.project(this._latlng).subtract([this._mRadius,0]));this._point=e.latLngToLayerPoint(this._latlng),this._radius=this._point.x-e.latLngToLayerPoint(c).x}this._updateBounds()}}),tn=Je.extend({options:{smoothFactor:1,noClip:!1},initialize:function(t,i){l(this,i),this._setLatLngs(t)},getLatLngs:function(){return this._latlngs},setLatLngs:function(t){return this._setLatLngs(t),this.redraw()},isEmpty:function(){return!this._latlngs.length},closestLayerPoint:function(t){for(var i,e,n=1/0,o=null,s=Rt,r=0,a=this._parts.length;r<a;r++)for(var h=this._parts[r],u=1,l=h.length;u<l;u++){var c=s(t,i=h[u-1],e=h[u],!0);c<n&&(n=c,o=s(t,i,e))}return o&&(o.distance=Math.sqrt(n)),o},getCenter:function(){if(!this._map)throw new Error("Must add layer to map before using getCenter()");var t,i,e,n,o,s,r,a=this._rings[0],h=a.length;if(!h)return null;for(t=0,i=0;t<h-1;t++)i+=a[t].distanceTo(a[t+1])/2;if(0===i)return this._map.layerPointToLatLng(a[0]);for(t=0,n=0;t<h-1;t++)if(o=a[t],s=a[t+1],e=o.distanceTo(s),(n+=e)>i)return r=(n-i)/e,this._map.layerPointToLatLng([s.x-r*(s.x-o.x),s.y-r*(s.y-o.y)])},getBounds:function(){return this._bounds},addLatLng:function(t,i){return i=i||this._defaultShape(),t=C(t),i.push(t),this._bounds.extend(t),this.redraw()},_setLatLngs:function(t){this._bounds=new T,this._latlngs=this._convertLatLngs(t)},_defaultShape:function(){return Dt(this._latlngs)?this._latlngs:this._latlngs[0]},_convertLatLngs:function(t){for(var i=[],e=Dt(t),n=0,o=t.length;n<o;n++)e?(i[n]=C(t[n]),this._bounds.extend(i[n])):i[n]=this._convertLatLngs(t[n]);return i},_project:function(){var t=new P;this._rings=[],this._projectLatlngs(this._latlngs,this._rings,t);var i=this._clickTolerance(),e=new x(i,i);this._bounds.isValid()&&t.isValid()&&(t.min._subtract(e),t.max._add(e),this._pxBounds=t)},_projectLatlngs:function(t,i,e){var n,o,s=t[0]instanceof M,r=t.length;if(s){for(o=[],n=0;n<r;n++)o[n]=this._map.latLngToLayerPoint(t[n]),e.extend(o[n]);i.push(o)}else for(n=0;n<r;n++)this._projectLatlngs(t[n],i,e)},_clipPoints:function(){var t=this._renderer._bounds;if(this._parts=[],this._pxBounds&&this._pxBounds.intersects(t))if(this.options.noClip)this._parts=this._rings;else{var i,e,n,o,s,r,a,h=this._parts;for(i=0,n=0,o=this._rings.length;i<o;i++)for(e=0,s=(a=this._rings[i]).length;e<s-1;e++)(r=At(a[e],a[e+1],t,e,!0))&&(h[n]=h[n]||[],h[n].push(r[0]),r[1]===a[e+1]&&e!==s-2||(h[n].push(r[1]),n++))}},_simplifyPoints:function(){for(var t=this._parts,i=this.options.smoothFactor,e=0,n=t.length;e<n;e++)t[e]=Ct(t[e],i)},_update:function(){this._map&&(this._clipPoints(),this._simplifyPoints(),this._updatePath())},_updatePath:function(){this._renderer._updatePoly(this)},_containsPoint:function(t,i){var e,n,o,s,r,a,h=this._clickTolerance();if(!this._pxBounds||!this._pxBounds.contains(t))return!1;for(e=0,s=this._parts.length;e<s;e++)for(n=0,o=(r=(a=this._parts[e]).length)-1;n<r;o=n++)if((i||0!==n)&&Zt(t,a[o],a[n])<=h)return!0;return!1}});tn._flat=Nt;var en=tn.extend({options:{fill:!0},isEmpty:function(){return!this._latlngs.length||!this._latlngs[0].length},getCenter:function(){if(!this._map)throw new Error("Must add layer to map before using getCenter()");var t,i,e,n,o,s,r,a,h,u=this._rings[0],l=u.length;if(!l)return null;for(s=r=a=0,t=0,i=l-1;t<l;i=t++)e=u[t],n=u[i],o=e.y*n.x-n.y*e.x,r+=(e.x+n.x)*o,a+=(e.y+n.y)*o,s+=3*o;return h=0===s?u[0]:[r/s,a/s],this._map.layerPointToLatLng(h)},_convertLatLngs:function(t){var i=tn.prototype._convertLatLngs.call(this,t),e=i.length;return e>=2&&i[0]instanceof M&&i[0].equals(i[e-1])&&i.pop(),i},_setLatLngs:function(t){tn.prototype._setLatLngs.call(this,t),Dt(this._latlngs)&&(this._latlngs=[this._latlngs])},_defaultShape:function(){return Dt(this._latlngs[0])?this._latlngs[0]:this._latlngs[0][0]},_clipPoints:function(){var t=this._renderer._bounds,i=this.options.weight,e=new x(i,i);if(t=new P(t.min.subtract(e),t.max.add(e)),this._parts=[],this._pxBounds&&this._pxBounds.intersects(t))if(this.options.noClip)this._parts=this._rings;else for(var n,o=0,s=this._rings.length;o<s;o++)(n=jt(this._rings[o],t,!0)).length&&this._parts.push(n)},_updatePath:function(){this._renderer._updatePoly(this,!0)},_containsPoint:function(t){var i,e,n,o,s,r,a,h,u=!1;if(!this._pxBounds.contains(t))return!1;for(o=0,a=this._parts.length;o<a;o++)for(s=0,r=(h=(i=this._parts[o]).length)-1;s<h;r=s++)e=i[s],n=i[r],e.y>t.y!=n.y>t.y&&t.x<(n.x-e.x)*(t.y-e.y)/(n.y-e.y)+e.x&&(u=!u);return u||tn.prototype._containsPoint.call(this,t,!0)}}),nn=qe.extend({initialize:function(t,i){l(this,i),this._layers={},t&&this.addData(t)},addData:function(t){var i,e,n,o=ei(t)?t:t.features;if(o){for(i=0,e=o.length;i<e;i++)((n=o[i]).geometries||n.geometry||n.features||n.coordinates)&&this.addData(n);return this}var s=this.options;if(s.filter&&!s.filter(t))return this;var r=Wt(t,s);return r?(r.feature=Gt(t),r.defaultOptions=r.options,this.resetStyle(r),s.onEachFeature&&s.onEachFeature(t,r),this.addLayer(r)):this},resetStyle:function(t){return t.options=i({},t.defaultOptions),this._setLayerStyle(t,this.options.style),this},setStyle:function(t){return this.eachLayer(function(i){this._setLayerStyle(i,t)},this)},_setLayerStyle:function(t,i){"function"==typeof i&&(i=i(t.feature)),t.setStyle&&t.setStyle(i)}}),on={toGeoJSON:function(t){return qt(this,{type:"Point",coordinates:Ut(this.getLatLng(),t)})}};Xe.include(on),Qe.include(on),$e.include(on),tn.include({toGeoJSON:function(t){var i=!Dt(this._latlngs),e=Vt(this._latlngs,i?1:0,!1,t);return qt(this,{type:(i?"Multi":"")+"LineString",coordinates:e})}}),en.include({toGeoJSON:function(t){var i=!Dt(this._latlngs),e=i&&!Dt(this._latlngs[0]),n=Vt(this._latlngs,e?2:i?1:0,!0,t);return i||(n=[n]),qt(this,{type:(e?"Multi":"")+"Polygon",coordinates:n})}}),Ve.include({toMultiPoint:function(t){var i=[];return this.eachLayer(function(e){i.push(e.toGeoJSON(t).geometry.coordinates)}),qt(this,{type:"MultiPoint",coordinates:i})},toGeoJSON:function(t){var i=this.feature&&this.feature.geometry&&this.feature.geometry.type;if("MultiPoint"===i)return this.toMultiPoint(t);var e="GeometryCollection"===i,n=[];return this.eachLayer(function(i){if(i.toGeoJSON){var o=i.toGeoJSON(t);if(e)n.push(o.geometry);else{var s=Gt(o);"FeatureCollection"===s.type?n.push.apply(n,s.features):n.push(s)}}}),e?qt(this,{geometries:n,type:"GeometryCollection"}):{type:"FeatureCollection",features:n}}});var sn=Kt,rn=Ue.extend({options:{opacity:1,alt:"",interactive:!1,crossOrigin:!1,errorOverlayUrl:"",zIndex:1,className:""},initialize:function(t,i,e){this._url=t,this._bounds=z(i),l(this,e)},onAdd:function(){this._image||(this._initImage(),this.options.opacity<1&&this._updateOpacity()),this.options.interactive&&(pt(this._image,"leaflet-interactive"),this.addInteractiveTarget(this._image)),this.getPane().appendChild(this._image),this._reset()},onRemove:function(){ut(this._image),this.options.interactive&&this.removeInteractiveTarget(this._image)},setOpacity:function(t){return this.options.opacity=t,this._image&&this._updateOpacity(),this},setStyle:function(t){return t.opacity&&this.setOpacity(t.opacity),this},bringToFront:function(){return this._map&&ct(this._image),this},bringToBack:function(){return this._map&&_t(this._image),this},setUrl:function(t){return this._url=t,this._image&&(this._image.src=t),this},setBounds:function(t){return this._bounds=z(t),this._map&&this._reset(),this},getEvents:function(){var t={zoom:this._reset,viewreset:this._reset};return this._zoomAnimated&&(t.zoomanim=this._animateZoom),t},setZIndex:function(t){return this.options.zIndex=t,this._updateZIndex(),this},getBounds:function(){return this._bounds},getElement:function(){return this._image},_initImage:function(){var t="IMG"===this._url.tagName,i=this._image=t?this._url:ht("img");pt(i,"leaflet-image-layer"),this._zoomAnimated&&pt(i,"leaflet-zoom-animated"),this.options.className&&pt(i,this.options.className),i.onselectstart=r,i.onmousemove=r,i.onload=e(this.fire,this,"load"),i.onerror=e(this._overlayOnError,this,"error"),this.options.crossOrigin&&(i.crossOrigin=""),this.options.zIndex&&this._updateZIndex(),t?this._url=i.src:(i.src=this._url,i.alt=this.options.alt)},_animateZoom:function(t){var i=this._map.getZoomScale(t.zoom),e=this._map._latLngBoundsToNewLayerBounds(this._bounds,t.zoom,t.center).min;wt(this._image,e,i)},_reset:function(){var t=this._image,i=new P(this._map.latLngToLayerPoint(this._bounds.getNorthWest()),this._map.latLngToLayerPoint(this._bounds.getSouthEast())),e=i.getSize();Lt(t,i.min),t.style.width=e.x+"px",t.style.height=e.y+"px"},_updateOpacity:function(){vt(this._image,this.options.opacity)},_updateZIndex:function(){this._image&&void 0!==this.options.zIndex&&null!==this.options.zIndex&&(this._image.style.zIndex=this.options.zIndex)},_overlayOnError:function(){this.fire("error");var t=this.options.errorOverlayUrl;t&&this._url!==t&&(this._url=t,this._image.src=t)}}),an=rn.extend({options:{autoplay:!0,loop:!0},_initImage:function(){var t="VIDEO"===this._url.tagName,i=this._image=t?this._url:ht("video");if(pt(i,"leaflet-image-layer"),this._zoomAnimated&&pt(i,"leaflet-zoom-animated"),i.onselectstart=r,i.onmousemove=r,i.onloadeddata=e(this.fire,this,"load"),t){for(var n=i.getElementsByTagName("source"),o=[],s=0;s<n.length;s++)o.push(n[s].src);this._url=n.length>0?o:[i.src]}else{ei(this._url)||(this._url=[this._url]),i.autoplay=!!this.options.autoplay,i.loop=!!this.options.loop;for(var a=0;a<this._url.length;a++){var h=ht("source");h.src=this._url[a],i.appendChild(h)}}}}),hn=Ue.extend({options:{offset:[0,7],className:"",pane:"popupPane"},initialize:function(t,i){l(this,t),this._source=i},onAdd:function(t){this._zoomAnimated=t._zoomAnimated,this._container||this._initLayout(),t._fadeAnimated&&vt(this._container,0),clearTimeout(this._removeTimeout),this.getPane().appendChild(this._container),this.update(),t._fadeAnimated&&vt(this._container,1),this.bringToFront()},onRemove:function(t){t._fadeAnimated?(vt(this._container,0),this._removeTimeout=setTimeout(e(ut,void 0,this._container),200)):ut(this._container)},getLatLng:function(){return this._latlng},setLatLng:function(t){return this._latlng=C(t),this._map&&(this._updatePosition(),this._adjustPan()),this},getContent:function(){return this._content},setContent:function(t){return this._content=t,this.update(),this},getElement:function(){return this._container},update:function(){this._map&&(this._container.style.visibility="hidden",this._updateContent(),this._updateLayout(),this._updatePosition(),this._container.style.visibility="",this._adjustPan())},getEvents:function(){var t={zoom:this._updatePosition,viewreset:this._updatePosition};return this._zoomAnimated&&(t.zoomanim=this._animateZoom),t},isOpen:function(){return!!this._map&&this._map.hasLayer(this)},bringToFront:function(){return this._map&&ct(this._container),this},bringToBack:function(){return this._map&&_t(this._container),this},_updateContent:function(){if(this._content){var t=this._contentNode,i="function"==typeof this._content?this._content(this._source||this):this._content;if("string"==typeof i)t.innerHTML=i;else{for(;t.hasChildNodes();)t.removeChild(t.firstChild);t.appendChild(i)}this.fire("contentupdate")}},_updatePosition:function(){if(this._map){var t=this._map.latLngToLayerPoint(this._latlng),i=w(this.options.offset),e=this._getAnchor();this._zoomAnimated?Lt(this._container,t.add(e)):i=i.add(t).add(e);var n=this._containerBottom=-i.y,o=this._containerLeft=-Math.round(this._containerWidth/2)+i.x;this._container.style.bottom=n+"px",this._container.style.left=o+"px"}},_getAnchor:function(){return[0,0]}}),un=hn.extend({options:{maxWidth:300,minWidth:50,maxHeight:null,autoPan:!0,autoPanPaddingTopLeft:null,autoPanPaddingBottomRight:null,autoPanPadding:[5,5],keepInView:!1,closeButton:!0,autoClose:!0,closeOnEscapeKey:!0,className:""},openOn:function(t){return t.openPopup(this),this},onAdd:function(t){hn.prototype.onAdd.call(this,t),t.fire("popupopen",{popup:this}),this._source&&(this._source.fire("popupopen",{popup:this},!0),this._source instanceof Je||this._source.on("preclick",Y))},onRemove:function(t){hn.prototype.onRemove.call(this,t),t.fire("popupclose",{popup:this}),this._source&&(this._source.fire("popupclose",{popup:this},!0),this._source instanceof Je||this._source.off("preclick",Y))},getEvents:function(){var t=hn.prototype.getEvents.call(this);return(void 0!==this.options.closeOnClick?this.options.closeOnClick:this._map.options.closePopupOnClick)&&(t.preclick=this._close),this.options.keepInView&&(t.moveend=this._adjustPan),t},_close:function(){this._map&&this._map.closePopup(this)},_initLayout:function(){var t="leaflet-popup",i=this._container=ht("div",t+" "+(this.options.className||"")+" leaflet-zoom-animated"),e=this._wrapper=ht("div",t+"-content-wrapper",i);if(this._contentNode=ht("div",t+"-content",e),J(e),X(this._contentNode),V(e,"contextmenu",Y),this._tipContainer=ht("div",t+"-tip-container",i),this._tip=ht("div",t+"-tip",this._tipContainer),this.options.closeButton){var n=this._closeButton=ht("a",t+"-close-button",i);n.href="#close",n.innerHTML="&#215;",V(n,"click",this._onCloseButtonClick,this)}},_updateLayout:function(){var t=this._contentNode,i=t.style;i.width="",i.whiteSpace="nowrap";var e=t.offsetWidth;e=Math.min(e,this.options.maxWidth),e=Math.max(e,this.options.minWidth),i.width=e+1+"px",i.whiteSpace="",i.height="";var n=t.offsetHeight,o=this.options.maxHeight;o&&n>o?(i.height=o+"px",pt(t,"leaflet-popup-scrolled")):mt(t,"leaflet-popup-scrolled"),this._containerWidth=this._container.offsetWidth},_animateZoom:function(t){var i=this._map._latLngToNewLayerPoint(this._latlng,t.zoom,t.center),e=this._getAnchor();Lt(this._container,i.add(e))},_adjustPan:function(){if(!(!this.options.autoPan||this._map._panAnim&&this._map._panAnim._inProgress)){var t=this._map,i=parseInt(at(this._container,"marginBottom"),10)||0,e=this._container.offsetHeight+i,n=this._containerWidth,o=new x(this._containerLeft,-e-this._containerBottom);o._add(Pt(this._container));var s=t.layerPointToContainerPoint(o),r=w(this.options.autoPanPadding),a=w(this.options.autoPanPaddingTopLeft||r),h=w(this.options.autoPanPaddingBottomRight||r),u=t.getSize(),l=0,c=0;s.x+n+h.x>u.x&&(l=s.x+n-u.x+h.x),s.x-l-a.x<0&&(l=s.x-a.x),s.y+e+h.y>u.y&&(c=s.y+e-u.y+h.y),s.y-c-a.y<0&&(c=s.y-a.y),(l||c)&&t.fire("autopanstart").panBy([l,c])}},_onCloseButtonClick:function(t){this._close(),Q(t)},_getAnchor:function(){return w(this._source&&this._source._getPopupAnchor?this._source._getPopupAnchor():[0,0])}});Le.mergeOptions({closePopupOnClick:!0}),Le.include({openPopup:function(t,i,e){return t instanceof un||(t=new un(e).setContent(t)),i&&t.setLatLng(i),this.hasLayer(t)?this:(this._popup&&this._popup.options.autoClose&&this.closePopup(),this._popup=t,this.addLayer(t))},closePopup:function(t){return t&&t!==this._popup||(t=this._popup,this._popup=null),t&&this.removeLayer(t),this}}),Ue.include({bindPopup:function(t,i){return t instanceof un?(l(t,i),this._popup=t,t._source=this):(this._popup&&!i||(this._popup=new un(i,this)),this._popup.setContent(t)),this._popupHandlersAdded||(this.on({click:this._openPopup,keypress:this._onKeyPress,remove:this.closePopup,move:this._movePopup}),this._popupHandlersAdded=!0),this},unbindPopup:function(){return this._popup&&(this.off({click:this._openPopup,keypress:this._onKeyPress,remove:this.closePopup,move:this._movePopup}),this._popupHandlersAdded=!1,this._popup=null),this},openPopup:function(t,i){if(t instanceof Ue||(i=t,t=this),t instanceof qe)for(var e in this._layers){t=this._layers[e];break}return i||(i=t.getCenter?t.getCenter():t.getLatLng()),this._popup&&this._map&&(this._popup._source=t,this._popup.update(),this._map.openPopup(this._popup,i)),this},closePopup:function(){return this._popup&&this._popup._close(),this},togglePopup:function(t){return this._popup&&(this._popup._map?this.closePopup():this.openPopup(t)),this},isPopupOpen:function(){return!!this._popup&&this._popup.isOpen()},setPopupContent:function(t){return this._popup&&this._popup.setContent(t),this},getPopup:function(){return this._popup},_openPopup:function(t){var i=t.layer||t.target;this._popup&&this._map&&(Q(t),i instanceof Je?this.openPopup(t.layer||t.target,t.latlng):this._map.hasLayer(this._popup)&&this._popup._source===i?this.closePopup():this.openPopup(i,t.latlng))},_movePopup:function(t){this._popup.setLatLng(t.latlng)},_onKeyPress:function(t){13===t.originalEvent.keyCode&&this._openPopup(t)}});var ln=hn.extend({options:{pane:"tooltipPane",offset:[0,0],direction:"auto",permanent:!1,sticky:!1,interactive:!1,opacity:.9},onAdd:function(t){hn.prototype.onAdd.call(this,t),this.setOpacity(this.options.opacity),t.fire("tooltipopen",{tooltip:this}),this._source&&this._source.fire("tooltipopen",{tooltip:this},!0)},onRemove:function(t){hn.prototype.onRemove.call(this,t),t.fire("tooltipclose",{tooltip:this}),this._source&&this._source.fire("tooltipclose",{tooltip:this},!0)},getEvents:function(){var t=hn.prototype.getEvents.call(this);return Vi&&!this.options.permanent&&(t.preclick=this._close),t},_close:function(){this._map&&this._map.closeTooltip(this)},_initLayout:function(){var t="leaflet-tooltip "+(this.options.className||"")+" leaflet-zoom-"+(this._zoomAnimated?"animated":"hide");this._contentNode=this._container=ht("div",t)},_updateLayout:function(){},_adjustPan:function(){},_setPosition:function(t){var i=this._map,e=this._container,n=i.latLngToContainerPoint(i.getCenter()),o=i.layerPointToContainerPoint(t),s=this.options.direction,r=e.offsetWidth,a=e.offsetHeight,h=w(this.options.offset),u=this._getAnchor();"top"===s?t=t.add(w(-r/2+h.x,-a+h.y+u.y,!0)):"bottom"===s?t=t.subtract(w(r/2-h.x,-h.y,!0)):"center"===s?t=t.subtract(w(r/2+h.x,a/2-u.y+h.y,!0)):"right"===s||"auto"===s&&o.x<n.x?(s="right",t=t.add(w(h.x+u.x,u.y-a/2+h.y,!0))):(s="left",t=t.subtract(w(r+u.x-h.x,a/2-u.y-h.y,!0))),mt(e,"leaflet-tooltip-right"),mt(e,"leaflet-tooltip-left"),mt(e,"leaflet-tooltip-top"),mt(e,"leaflet-tooltip-bottom"),pt(e,"leaflet-tooltip-"+s),Lt(e,t)},_updatePosition:function(){var t=this._map.latLngToLayerPoint(this._latlng);this._setPosition(t)},setOpacity:function(t){this.options.opacity=t,this._container&&vt(this._container,t)},_animateZoom:function(t){var i=this._map._latLngToNewLayerPoint(this._latlng,t.zoom,t.center);this._setPosition(i)},_getAnchor:function(){return w(this._source&&this._source._getTooltipAnchor&&!this.options.sticky?this._source._getTooltipAnchor():[0,0])}});Le.include({openTooltip:function(t,i,e){return t instanceof ln||(t=new ln(e).setContent(t)),i&&t.setLatLng(i),this.hasLayer(t)?this:this.addLayer(t)},closeTooltip:function(t){return t&&this.removeLayer(t),this}}),Ue.include({bindTooltip:function(t,i){return t instanceof ln?(l(t,i),this._tooltip=t,t._source=this):(this._tooltip&&!i||(this._tooltip=new ln(i,this)),this._tooltip.setContent(t)),this._initTooltipInteractions(),this._tooltip.options.permanent&&this._map&&this._map.hasLayer(this)&&this.openTooltip(),this},unbindTooltip:function(){return this._tooltip&&(this._initTooltipInteractions(!0),this.closeTooltip(),this._tooltip=null),this},_initTooltipInteractions:function(t){if(t||!this._tooltipHandlersAdded){var i=t?"off":"on",e={remove:this.closeTooltip,move:this._moveTooltip};this._tooltip.options.permanent?e.add=this._openTooltip:(e.mouseover=this._openTooltip,e.mouseout=this.closeTooltip,this._tooltip.options.sticky&&(e.mousemove=this._moveTooltip),Vi&&(e.click=this._openTooltip)),this[i](e),this._tooltipHandlersAdded=!t}},openTooltip:function(t,i){if(t instanceof Ue||(i=t,t=this),t instanceof qe)for(var e in this._layers){t=this._layers[e];break}return i||(i=t.getCenter?t.getCenter():t.getLatLng()),this._tooltip&&this._map&&(this._tooltip._source=t,this._tooltip.update(),this._map.openTooltip(this._tooltip,i),this._tooltip.options.interactive&&this._tooltip._container&&(pt(this._tooltip._container,"leaflet-clickable"),this.addInteractiveTarget(this._tooltip._container))),this},closeTooltip:function(){return this._tooltip&&(this._tooltip._close(),this._tooltip.options.interactive&&this._tooltip._container&&(mt(this._tooltip._container,"leaflet-clickable"),this.removeInteractiveTarget(this._tooltip._container))),this},toggleTooltip:function(t){return this._tooltip&&(this._tooltip._map?this.closeTooltip():this.openTooltip(t)),this},isTooltipOpen:function(){return this._tooltip.isOpen()},setTooltipContent:function(t){return this._tooltip&&this._tooltip.setContent(t),this},getTooltip:function(){return this._tooltip},_openTooltip:function(t){var i=t.layer||t.target;this._tooltip&&this._map&&this.openTooltip(i,this._tooltip.options.sticky?t.latlng:void 0)},_moveTooltip:function(t){var i,e,n=t.latlng;this._tooltip.options.sticky&&t.originalEvent&&(i=this._map.mouseEventToContainerPoint(t.originalEvent),e=this._map.containerPointToLayerPoint(i),n=this._map.layerPointToLatLng(e)),this._tooltip.setLatLng(n)}});var cn=Ge.extend({options:{iconSize:[12,12],html:!1,bgPos:null,className:"leaflet-div-icon"},createIcon:function(t){var i=t&&"DIV"===t.tagName?t:document.createElement("div"),e=this.options;if(i.innerHTML=!1!==e.html?e.html:"",e.bgPos){var n=w(e.bgPos);i.style.backgroundPosition=-n.x+"px "+-n.y+"px"}return this._setIconStyles(i,"icon"),i},createShadow:function(){return null}});Ge.Default=Ke;var _n=Ue.extend({options:{tileSize:256,opacity:1,updateWhenIdle:ji,updateWhenZooming:!0,updateInterval:200,zIndex:1,bounds:null,minZoom:0,maxZoom:void 0,maxNativeZoom:void 0,minNativeZoom:void 0,noWrap:!1,pane:"tilePane",className:"",keepBuffer:2},initialize:function(t){l(this,t)},onAdd:function(){this._initContainer(),this._levels={},this._tiles={},this._resetView(),this._update()},beforeAdd:function(t){t._addZoomLimit(this)},onRemove:function(t){this._removeAllTiles(),ut(this._container),t._removeZoomLimit(this),this._container=null,this._tileZoom=void 0},bringToFront:function(){return this._map&&(ct(this._container),this._setAutoZIndex(Math.max)),this},bringToBack:function(){return this._map&&(_t(this._container),this._setAutoZIndex(Math.min)),this},getContainer:function(){return this._container},setOpacity:function(t){return this.options.opacity=t,this._updateOpacity(),this},setZIndex:function(t){return this.options.zIndex=t,this._updateZIndex(),this},isLoading:function(){return this._loading},redraw:function(){return this._map&&(this._removeAllTiles(),this._update()),this},getEvents:function(){var t={viewprereset:this._invalidateAll,viewreset:this._resetView,zoom:this._resetView,moveend:this._onMoveEnd};return this.options.updateWhenIdle||(this._onMove||(this._onMove=o(this._onMoveEnd,this.options.updateInterval,this)),t.move=this._onMove),this._zoomAnimated&&(t.zoomanim=this._animateZoom),t},createTile:function(){return document.createElement("div")},getTileSize:function(){var t=this.options.tileSize;return t instanceof x?t:new x(t,t)},_updateZIndex:function(){this._container&&void 0!==this.options.zIndex&&null!==this.options.zIndex&&(this._container.style.zIndex=this.options.zIndex)},_setAutoZIndex:function(t){for(var i,e=this.getPane().children,n=-t(-1/0,1/0),o=0,s=e.length;o<s;o++)i=e[o].style.zIndex,e[o]!==this._container&&i&&(n=t(n,+i));isFinite(n)&&(this.options.zIndex=n+t(-1,1),this._updateZIndex())},_updateOpacity:function(){if(this._map&&!Li){vt(this._container,this.options.opacity);var t=+new Date,i=!1,e=!1;for(var n in this._tiles){var o=this._tiles[n];if(o.current&&o.loaded){var s=Math.min(1,(t-o.loaded)/200);vt(o.el,s),s<1?i=!0:(o.active?e=!0:this._onOpaqueTile(o),o.active=!0)}}e&&!this._noPrune&&this._pruneTiles(),i&&(g(this._fadeFrame),this._fadeFrame=f(this._updateOpacity,this))}},_onOpaqueTile:r,_initContainer:function(){this._container||(this._container=ht("div","leaflet-layer "+(this.options.className||"")),this._updateZIndex(),this.options.opacity<1&&this._updateOpacity(),this.getPane().appendChild(this._container))},_updateLevels:function(){var t=this._tileZoom,i=this.options.maxZoom;if(void 0!==t){for(var e in this._levels)this._levels[e].el.children.length||e===t?(this._levels[e].el.style.zIndex=i-Math.abs(t-e),this._onUpdateLevel(e)):(ut(this._levels[e].el),this._removeTilesAtZoom(e),this._onRemoveLevel(e),delete this._levels[e]);var n=this._levels[t],o=this._map;return n||((n=this._levels[t]={}).el=ht("div","leaflet-tile-container leaflet-zoom-animated",this._container),n.el.style.zIndex=i,n.origin=o.project(o.unproject(o.getPixelOrigin()),t).round(),n.zoom=t,this._setZoomTransform(n,o.getCenter(),o.getZoom()),n.el.offsetWidth,this._onCreateLevel(n)),this._level=n,n}},_onUpdateLevel:r,_onRemoveLevel:r,_onCreateLevel:r,_pruneTiles:function(){if(this._map){var t,i,e=this._map.getZoom();if(e>this.options.maxZoom||e<this.options.minZoom)this._removeAllTiles();else{for(t in this._tiles)(i=this._tiles[t]).retain=i.current;for(t in this._tiles)if((i=this._tiles[t]).current&&!i.active){var n=i.coords;this._retainParent(n.x,n.y,n.z,n.z-5)||this._retainChildren(n.x,n.y,n.z,n.z+2)}for(t in this._tiles)this._tiles[t].retain||this._removeTile(t)}}},_removeTilesAtZoom:function(t){for(var i in this._tiles)this._tiles[i].coords.z===t&&this._removeTile(i)},_removeAllTiles:function(){for(var t in this._tiles)this._removeTile(t)},_invalidateAll:function(){for(var t in this._levels)ut(this._levels[t].el),this._onRemoveLevel(t),delete this._levels[t];this._removeAllTiles(),this._tileZoom=void 0},_retainParent:function(t,i,e,n){var o=Math.floor(t/2),s=Math.floor(i/2),r=e-1,a=new x(+o,+s);a.z=+r;var h=this._tileCoordsToKey(a),u=this._tiles[h];return u&&u.active?(u.retain=!0,!0):(u&&u.loaded&&(u.retain=!0),r>n&&this._retainParent(o,s,r,n))},_retainChildren:function(t,i,e,n){for(var o=2*t;o<2*t+2;o++)for(var s=2*i;s<2*i+2;s++){var r=new x(o,s);r.z=e+1;var a=this._tileCoordsToKey(r),h=this._tiles[a];h&&h.active?h.retain=!0:(h&&h.loaded&&(h.retain=!0),e+1<n&&this._retainChildren(o,s,e+1,n))}},_resetView:function(t){var i=t&&(t.pinch||t.flyTo);this._setView(this._map.getCenter(),this._map.getZoom(),i,i)},_animateZoom:function(t){this._setView(t.center,t.zoom,!0,t.noUpdate)},_clampZoom:function(t){var i=this.options;return void 0!==i.minNativeZoom&&t<i.minNativeZoom?i.minNativeZoom:void 0!==i.maxNativeZoom&&i.maxNativeZoom<t?i.maxNativeZoom:t},_setView:function(t,i,e,n){var o=this._clampZoom(Math.round(i));(void 0!==this.options.maxZoom&&o>this.options.maxZoom||void 0!==this.options.minZoom&&o<this.options.minZoom)&&(o=void 0);var s=this.options.updateWhenZooming&&o!==this._tileZoom;n&&!s||(this._tileZoom=o,this._abortLoading&&this._abortLoading(),this._updateLevels(),this._resetGrid(),void 0!==o&&this._update(t),e||this._pruneTiles(),this._noPrune=!!e),this._setZoomTransforms(t,i)},_setZoomTransforms:function(t,i){for(var e in this._levels)this._setZoomTransform(this._levels[e],t,i)},_setZoomTransform:function(t,i,e){var n=this._map.getZoomScale(e,t.zoom),o=t.origin.multiplyBy(n).subtract(this._map._getNewPixelOrigin(i,e)).round();Ni?wt(t.el,o,n):Lt(t.el,o)},_resetGrid:function(){var t=this._map,i=t.options.crs,e=this._tileSize=this.getTileSize(),n=this._tileZoom,o=this._map.getPixelWorldBounds(this._tileZoom);o&&(this._globalTileRange=this._pxBoundsToTileRange(o)),this._wrapX=i.wrapLng&&!this.options.noWrap&&[Math.floor(t.project([0,i.wrapLng[0]],n).x/e.x),Math.ceil(t.project([0,i.wrapLng[1]],n).x/e.y)],this._wrapY=i.wrapLat&&!this.options.noWrap&&[Math.floor(t.project([i.wrapLat[0],0],n).y/e.x),Math.ceil(t.project([i.wrapLat[1],0],n).y/e.y)]},_onMoveEnd:function(){this._map&&!this._map._animatingZoom&&this._update()},_getTiledPixelBounds:function(t){var i=this._map,e=i._animatingZoom?Math.max(i._animateToZoom,i.getZoom()):i.getZoom(),n=i.getZoomScale(e,this._tileZoom),o=i.project(t,this._tileZoom).floor(),s=i.getSize().divideBy(2*n);return new P(o.subtract(s),o.add(s))},_update:function(t){var i=this._map;if(i){var e=this._clampZoom(i.getZoom());if(void 0===t&&(t=i.getCenter()),void 0!==this._tileZoom){var n=this._getTiledPixelBounds(t),o=this._pxBoundsToTileRange(n),s=o.getCenter(),r=[],a=this.options.keepBuffer,h=new P(o.getBottomLeft().subtract([a,-a]),o.getTopRight().add([a,-a]));if(!(isFinite(o.min.x)&&isFinite(o.min.y)&&isFinite(o.max.x)&&isFinite(o.max.y)))throw new Error("Attempted to load an infinite number of tiles");for(var u in this._tiles){var l=this._tiles[u].coords;l.z===this._tileZoom&&h.contains(new x(l.x,l.y))||(this._tiles[u].current=!1)}if(Math.abs(e-this._tileZoom)>1)this._setView(t,e);else{for(var c=o.min.y;c<=o.max.y;c++)for(var _=o.min.x;_<=o.max.x;_++){var d=new x(_,c);if(d.z=this._tileZoom,this._isValidTile(d)){var p=this._tiles[this._tileCoordsToKey(d)];p?p.current=!0:r.push(d)}}if(r.sort(function(t,i){return t.distanceTo(s)-i.distanceTo(s)}),0!==r.length){this._loading||(this._loading=!0,this.fire("loading"));var m=document.createDocumentFragment();for(_=0;_<r.length;_++)this._addTile(r[_],m);this._level.el.appendChild(m)}}}}},_isValidTile:function(t){var i=this._map.options.crs;if(!i.infinite){var e=this._globalTileRange;if(!i.wrapLng&&(t.x<e.min.x||t.x>e.max.x)||!i.wrapLat&&(t.y<e.min.y||t.y>e.max.y))return!1}if(!this.options.bounds)return!0;var n=this._tileCoordsToBounds(t);return z(this.options.bounds).overlaps(n)},_keyToBounds:function(t){return this._tileCoordsToBounds(this._keyToTileCoords(t))},_tileCoordsToNwSe:function(t){var i=this._map,e=this.getTileSize(),n=t.scaleBy(e),o=n.add(e);return[i.unproject(n,t.z),i.unproject(o,t.z)]},_tileCoordsToBounds:function(t){var i=this._tileCoordsToNwSe(t),e=new T(i[0],i[1]);return this.options.noWrap||(e=this._map.wrapLatLngBounds(e)),e},_tileCoordsToKey:function(t){return t.x+":"+t.y+":"+t.z},_keyToTileCoords:function(t){var i=t.split(":"),e=new x(+i[0],+i[1]);return e.z=+i[2],e},_removeTile:function(t){var i=this._tiles[t];i&&(Ci||i.el.setAttribute("src",ni),ut(i.el),delete this._tiles[t],this.fire("tileunload",{tile:i.el,coords:this._keyToTileCoords(t)}))},_initTile:function(t){pt(t,"leaflet-tile");var i=this.getTileSize();t.style.width=i.x+"px",t.style.height=i.y+"px",t.onselectstart=r,t.onmousemove=r,Li&&this.options.opacity<1&&vt(t,this.options.opacity),Ti&&!zi&&(t.style.WebkitBackfaceVisibility="hidden")},_addTile:function(t,i){var n=this._getTilePos(t),o=this._tileCoordsToKey(t),s=this.createTile(this._wrapCoords(t),e(this._tileReady,this,t));this._initTile(s),this.createTile.length<2&&f(e(this._tileReady,this,t,null,s)),Lt(s,n),this._tiles[o]={el:s,coords:t,current:!0},i.appendChild(s),this.fire("tileloadstart",{tile:s,coords:t})},_tileReady:function(t,i,n){if(this._map){i&&this.fire("tileerror",{error:i,tile:n,coords:t});var o=this._tileCoordsToKey(t);(n=this._tiles[o])&&(n.loaded=+new Date,this._map._fadeAnimated?(vt(n.el,0),g(this._fadeFrame),this._fadeFrame=f(this._updateOpacity,this)):(n.active=!0,this._pruneTiles()),i||(pt(n.el,"leaflet-tile-loaded"),this.fire("tileload",{tile:n.el,coords:t})),this._noTilesToLoad()&&(this._loading=!1,this.fire("load"),Li||!this._map._fadeAnimated?f(this._pruneTiles,this):setTimeout(e(this._pruneTiles,this),250)))}},_getTilePos:function(t){return t.scaleBy(this.getTileSize()).subtract(this._level.origin)},_wrapCoords:function(t){var i=new x(this._wrapX?s(t.x,this._wrapX):t.x,this._wrapY?s(t.y,this._wrapY):t.y);return i.z=t.z,i},_pxBoundsToTileRange:function(t){var i=this.getTileSize();return new P(t.min.unscaleBy(i).floor(),t.max.unscaleBy(i).ceil().subtract([1,1]))},_noTilesToLoad:function(){for(var t in this._tiles)if(!this._tiles[t].loaded)return!1;return!0}}),dn=_n.extend({options:{minZoom:0,maxZoom:18,subdomains:"abc",errorTileUrl:"",zoomOffset:0,tms:!1,zoomReverse:!1,detectRetina:!1,crossOrigin:!1},initialize:function(t,i){this._url=t,(i=l(this,i)).detectRetina&&Ki&&i.maxZoom>0&&(i.tileSize=Math.floor(i.tileSize/2),i.zoomReverse?(i.zoomOffset--,i.minZoom++):(i.zoomOffset++,i.maxZoom--),i.minZoom=Math.max(0,i.minZoom)),"string"==typeof i.subdomains&&(i.subdomains=i.subdomains.split("")),Ti||this.on("tileunload",this._onTileRemove)},setUrl:function(t,i){return this._url=t,i||this.redraw(),this},createTile:function(t,i){var n=document.createElement("img");return V(n,"load",e(this._tileOnLoad,this,i,n)),V(n,"error",e(this._tileOnError,this,i,n)),this.options.crossOrigin&&(n.crossOrigin=""),n.alt="",n.setAttribute("role","presentation"),n.src=this.getTileUrl(t),n},getTileUrl:function(t){var e={r:Ki?"@2x":"",s:this._getSubdomain(t),x:t.x,y:t.y,z:this._getZoomForUrl()};if(this._map&&!this._map.options.crs.infinite){var n=this._globalTileRange.max.y-t.y;this.options.tms&&(e.y=n),e["-y"]=n}return _(this._url,i(e,this.options))},_tileOnLoad:function(t,i){Li?setTimeout(e(t,this,null,i),0):t(null,i)},_tileOnError:function(t,i,e){var n=this.options.errorTileUrl;n&&i.getAttribute("src")!==n&&(i.src=n),t(e,i)},_onTileRemove:function(t){t.tile.onload=null},_getZoomForUrl:function(){var t=this._tileZoom,i=this.options.maxZoom,e=this.options.zoomReverse,n=this.options.zoomOffset;return e&&(t=i-t),t+n},_getSubdomain:function(t){var i=Math.abs(t.x+t.y)%this.options.subdomains.length;return this.options.subdomains[i]},_abortLoading:function(){var t,i;for(t in this._tiles)this._tiles[t].coords.z!==this._tileZoom&&((i=this._tiles[t].el).onload=r,i.onerror=r,i.complete||(i.src=ni,ut(i),delete this._tiles[t]))}}),pn=dn.extend({defaultWmsParams:{service:"WMS",request:"GetMap",layers:"",styles:"",format:"image/jpeg",transparent:!1,version:"1.1.1"},options:{crs:null,uppercase:!1},initialize:function(t,e){this._url=t;var n=i({},this.defaultWmsParams);for(var o in e)o in this.options||(n[o]=e[o]);var s=(e=l(this,e)).detectRetina&&Ki?2:1,r=this.getTileSize();n.width=r.x*s,n.height=r.y*s,this.wmsParams=n},onAdd:function(t){this._crs=this.options.crs||t.options.crs,this._wmsVersion=parseFloat(this.wmsParams.version);var i=this._wmsVersion>=1.3?"crs":"srs";this.wmsParams[i]=this._crs.code,dn.prototype.onAdd.call(this,t)},getTileUrl:function(t){var i=this._tileCoordsToNwSe(t),e=this._crs,n=b(e.project(i[0]),e.project(i[1])),o=n.min,s=n.max,r=(this._wmsVersion>=1.3&&this._crs===He?[o.y,o.x,s.y,s.x]:[o.x,o.y,s.x,s.y]).join(","),a=L.TileLayer.prototype.getTileUrl.call(this,t);return a+c(this.wmsParams,a,this.options.uppercase)+(this.options.uppercase?"&BBOX=":"&bbox=")+r},setParams:function(t,e){return i(this.wmsParams,t),e||this.redraw(),this}});dn.WMS=pn,Yt.wms=function(t,i){return new pn(t,i)};var mn=Ue.extend({options:{padding:.1,tolerance:0},initialize:function(t){l(this,t),n(this),this._layers=this._layers||{}},onAdd:function(){this._container||(this._initContainer(),this._zoomAnimated&&pt(this._container,"leaflet-zoom-animated")),this.getPane().appendChild(this._container),this._update(),this.on("update",this._updatePaths,this)},onRemove:function(){this.off("update",this._updatePaths,this),this._destroyContainer()},getEvents:function(){var t={viewreset:this._reset,zoom:this._onZoom,moveend:this._update,zoomend:this._onZoomEnd};return this._zoomAnimated&&(t.zoomanim=this._onAnimZoom),t},_onAnimZoom:function(t){this._updateTransform(t.center,t.zoom)},_onZoom:function(){this._updateTransform(this._map.getCenter(),this._map.getZoom())},_updateTransform:function(t,i){var e=this._map.getZoomScale(i,this._zoom),n=Pt(this._container),o=this._map.getSize().multiplyBy(.5+this.options.padding),s=this._map.project(this._center,i),r=this._map.project(t,i).subtract(s),a=o.multiplyBy(-e).add(n).add(o).subtract(r);Ni?wt(this._container,a,e):Lt(this._container,a)},_reset:function(){this._update(),this._updateTransform(this._center,this._zoom);for(var t in this._layers)this._layers[t]._reset()},_onZoomEnd:function(){for(var t in this._layers)this._layers[t]._project()},_updatePaths:function(){for(var t in this._layers)this._layers[t]._update()},_update:function(){var t=this.options.padding,i=this._map.getSize(),e=this._map.containerPointToLayerPoint(i.multiplyBy(-t)).round();this._bounds=new P(e,e.add(i.multiplyBy(1+2*t)).round()),this._center=this._map.getCenter(),this._zoom=this._map.getZoom()}}),fn=mn.extend({getEvents:function(){var t=mn.prototype.getEvents.call(this);return t.viewprereset=this._onViewPreReset,t},_onViewPreReset:function(){this._postponeUpdatePaths=!0},onAdd:function(){mn.prototype.onAdd.call(this),this._draw()},_initContainer:function(){var t=this._container=document.createElement("canvas");V(t,"mousemove",o(this._onMouseMove,32,this),this),V(t,"click dblclick mousedown mouseup contextmenu",this._onClick,this),V(t,"mouseout",this._handleMouseOut,this),this._ctx=t.getContext("2d")},_destroyContainer:function(){delete this._ctx,ut(this._container),q(this._container),delete this._container},_updatePaths:function(){if(!this._postponeUpdatePaths){this._redrawBounds=null;for(var t in this._layers)this._layers[t]._update();this._redraw()}},_update:function(){if(!this._map._animatingZoom||!this._bounds){this._drawnLayers={},mn.prototype._update.call(this);var t=this._bounds,i=this._container,e=t.getSize(),n=Ki?2:1;Lt(i,t.min),i.width=n*e.x,i.height=n*e.y,i.style.width=e.x+"px",i.style.height=e.y+"px",Ki&&this._ctx.scale(2,2),this._ctx.translate(-t.min.x,-t.min.y),this.fire("update")}},_reset:function(){mn.prototype._reset.call(this),this._postponeUpdatePaths&&(this._postponeUpdatePaths=!1,this._updatePaths())},_initPath:function(t){this._updateDashArray(t),this._layers[n(t)]=t;var i=t._order={layer:t,prev:this._drawLast,next:null};this._drawLast&&(this._drawLast.next=i),this._drawLast=i,this._drawFirst=this._drawFirst||this._drawLast},_addPath:function(t){this._requestRedraw(t)},_removePath:function(t){var i=t._order,e=i.next,n=i.prev;e?e.prev=n:this._drawLast=n,n?n.next=e:this._drawFirst=e,delete t._order,delete this._layers[L.stamp(t)],this._requestRedraw(t)},_updatePath:function(t){this._extendRedrawBounds(t),t._project(),t._update(),this._requestRedraw(t)},_updateStyle:function(t){this._updateDashArray(t),this._requestRedraw(t)},_updateDashArray:function(t){if(t.options.dashArray){var i,e=t.options.dashArray.split(","),n=[];for(i=0;i<e.length;i++)n.push(Number(e[i]));t.options._dashArray=n}},_requestRedraw:function(t){this._map&&(this._extendRedrawBounds(t),this._redrawRequest=this._redrawRequest||f(this._redraw,this))},_extendRedrawBounds:function(t){if(t._pxBounds){var i=(t.options.weight||0)+1;this._redrawBounds=this._redrawBounds||new P,this._redrawBounds.extend(t._pxBounds.min.subtract([i,i])),this._redrawBounds.extend(t._pxBounds.max.add([i,i]))}},_redraw:function(){this._redrawRequest=null,this._redrawBounds&&(this._redrawBounds.min._floor(),this._redrawBounds.max._ceil()),this._clear(),this._draw(),this._redrawBounds=null},_clear:function(){var t=this._redrawBounds;if(t){var i=t.getSize();this._ctx.clearRect(t.min.x,t.min.y,i.x,i.y)}else this._ctx.clearRect(0,0,this._container.width,this._container.height)},_draw:function(){var t,i=this._redrawBounds;if(this._ctx.save(),i){var e=i.getSize();this._ctx.beginPath(),this._ctx.rect(i.min.x,i.min.y,e.x,e.y),this._ctx.clip()}this._drawing=!0;for(var n=this._drawFirst;n;n=n.next)t=n.layer,(!i||t._pxBounds&&t._pxBounds.intersects(i))&&t._updatePath();this._drawing=!1,this._ctx.restore()},_updatePoly:function(t,i){if(this._drawing){var e,n,o,s,r=t._parts,a=r.length,h=this._ctx;if(a){for(this._drawnLayers[t._leaflet_id]=t,h.beginPath(),e=0;e<a;e++){for(n=0,o=r[e].length;n<o;n++)s=r[e][n],h[n?"lineTo":"moveTo"](s.x,s.y);i&&h.closePath()}this._fillStroke(h,t)}}},_updateCircle:function(t){if(this._drawing&&!t._empty()){var i=t._point,e=this._ctx,n=Math.max(Math.round(t._radius),1),o=(Math.max(Math.round(t._radiusY),1)||n)/n;this._drawnLayers[t._leaflet_id]=t,1!==o&&(e.save(),e.scale(1,o)),e.beginPath(),e.arc(i.x,i.y/o,n,0,2*Math.PI,!1),1!==o&&e.restore(),this._fillStroke(e,t)}},_fillStroke:function(t,i){var e=i.options;e.fill&&(t.globalAlpha=e.fillOpacity,t.fillStyle=e.fillColor||e.color,t.fill(e.fillRule||"evenodd")),e.stroke&&0!==e.weight&&(t.setLineDash&&t.setLineDash(i.options&&i.options._dashArray||[]),t.globalAlpha=e.opacity,t.lineWidth=e.weight,t.strokeStyle=e.color,t.lineCap=e.lineCap,t.lineJoin=e.lineJoin,t.stroke())},_onClick:function(t){for(var i,e,n=this._map.mouseEventToLayerPoint(t),o=this._drawFirst;o;o=o.next)(i=o.layer).options.interactive&&i._containsPoint(n)&&!this._map._draggableMoved(i)&&(e=i);e&&(et(t),this._fireEvent([e],t))},_onMouseMove:function(t){if(this._map&&!this._map.dragging.moving()&&!this._map._animatingZoom){var i=this._map.mouseEventToLayerPoint(t);this._handleMouseHover(t,i)}},_handleMouseOut:function(t){var i=this._hoveredLayer;i&&(mt(this._container,"leaflet-interactive"),this._fireEvent([i],t,"mouseout"),this._hoveredLayer=null)},_handleMouseHover:function(t,i){for(var e,n,o=this._drawFirst;o;o=o.next)(e=o.layer).options.interactive&&e._containsPoint(i)&&(n=e);n!==this._hoveredLayer&&(this._handleMouseOut(t),n&&(pt(this._container,"leaflet-interactive"),this._fireEvent([n],t,"mouseover"),this._hoveredLayer=n)),this._hoveredLayer&&this._fireEvent([this._hoveredLayer],t)},_fireEvent:function(t,i,e){this._map._fireDOMEvent(i,e||i.type,t)},_bringToFront:function(t){var i=t._order,e=i.next,n=i.prev;e&&(e.prev=n,n?n.next=e:e&&(this._drawFirst=e),i.prev=this._drawLast,this._drawLast.next=i,i.next=null,this._drawLast=i,this._requestRedraw(t))},_bringToBack:function(t){var i=t._order,e=i.next,n=i.prev;n&&(n.next=e,e?e.prev=n:n&&(this._drawLast=n),i.prev=null,i.next=this._drawFirst,this._drawFirst.prev=i,this._drawFirst=i,this._requestRedraw(t))}}),gn=function(){try{return document.namespaces.add("lvml","urn:schemas-microsoft-com:vml"),function(t){return document.createElement("<lvml:"+t+' class="lvml">')}}catch(t){return function(t){return document.createElement("<"+t+' xmlns="urn:schemas-microsoft.com:vml" class="lvml">')}}}(),vn={_initContainer:function(){this._container=ht("div","leaflet-vml-container")},_update:function(){this._map._animatingZoom||(mn.prototype._update.call(this),this.fire("update"))},_initPath:function(t){var i=t._container=gn("shape");pt(i,"leaflet-vml-shape "+(this.options.className||"")),i.coordsize="1 1",t._path=gn("path"),i.appendChild(t._path),this._updateStyle(t),this._layers[n(t)]=t},_addPath:function(t){var i=t._container;this._container.appendChild(i),t.options.interactive&&t.addInteractiveTarget(i)},_removePath:function(t){var i=t._container;ut(i),t.removeInteractiveTarget(i),delete this._layers[n(t)]},_updateStyle:function(t){var i=t._stroke,e=t._fill,n=t.options,o=t._container;o.stroked=!!n.stroke,o.filled=!!n.fill,n.stroke?(i||(i=t._stroke=gn("stroke")),o.appendChild(i),i.weight=n.weight+"px",i.color=n.color,i.opacity=n.opacity,n.dashArray?i.dashStyle=ei(n.dashArray)?n.dashArray.join(" "):n.dashArray.replace(/( *, *)/g," "):i.dashStyle="",i.endcap=n.lineCap.replace("butt","flat"),i.joinstyle=n.lineJoin):i&&(o.removeChild(i),t._stroke=null),n.fill?(e||(e=t._fill=gn("fill")),o.appendChild(e),e.color=n.fillColor||n.color,e.opacity=n.fillOpacity):e&&(o.removeChild(e),t._fill=null)},_updateCircle:function(t){var i=t._point.round(),e=Math.round(t._radius),n=Math.round(t._radiusY||e);this._setPath(t,t._empty()?"M0 0":"AL "+i.x+","+i.y+" "+e+","+n+" 0,23592600")},_setPath:function(t,i){t._path.v=i},_bringToFront:function(t){ct(t._container)},_bringToBack:function(t){_t(t._container)}},yn=Ji?gn:E,xn=mn.extend({getEvents:function(){var t=mn.prototype.getEvents.call(this);return t.zoomstart=this._onZoomStart,t},_initContainer:function(){this._container=yn("svg"),this._container.setAttribute("pointer-events","none"),this._rootGroup=yn("g"),this._container.appendChild(this._rootGroup)},_destroyContainer:function(){ut(this._container),q(this._container),delete this._container,delete this._rootGroup,delete this._svgSize},_onZoomStart:function(){this._update()},_update:function(){if(!this._map._animatingZoom||!this._bounds){mn.prototype._update.call(this);var t=this._bounds,i=t.getSize(),e=this._container;this._svgSize&&this._svgSize.equals(i)||(this._svgSize=i,e.setAttribute("width",i.x),e.setAttribute("height",i.y)),Lt(e,t.min),e.setAttribute("viewBox",[t.min.x,t.min.y,i.x,i.y].join(" ")),this.fire("update")}},_initPath:function(t){var i=t._path=yn("path");t.options.className&&pt(i,t.options.className),t.options.interactive&&pt(i,"leaflet-interactive"),this._updateStyle(t),this._layers[n(t)]=t},_addPath:function(t){this._rootGroup||this._initContainer(),this._rootGroup.appendChild(t._path),t.addInteractiveTarget(t._path)},_removePath:function(t){ut(t._path),t.removeInteractiveTarget(t._path),delete this._layers[n(t)]},_updatePath:function(t){t._project(),t._update()},_updateStyle:function(t){var i=t._path,e=t.options;i&&(e.stroke?(i.setAttribute("stroke",e.color),i.setAttribute("stroke-opacity",e.opacity),i.setAttribute("stroke-width",e.weight),i.setAttribute("stroke-linecap",e.lineCap),i.setAttribute("stroke-linejoin",e.lineJoin),e.dashArray?i.setAttribute("stroke-dasharray",e.dashArray):i.removeAttribute("stroke-dasharray"),e.dashOffset?i.setAttribute("stroke-dashoffset",e.dashOffset):i.removeAttribute("stroke-dashoffset")):i.setAttribute("stroke","none"),e.fill?(i.setAttribute("fill",e.fillColor||e.color),i.setAttribute("fill-opacity",e.fillOpacity),i.setAttribute("fill-rule",e.fillRule||"evenodd")):i.setAttribute("fill","none"))},_updatePoly:function(t,i){this._setPath(t,k(t._parts,i))},_updateCircle:function(t){var i=t._point,e=Math.max(Math.round(t._radius),1),n="a"+e+","+(Math.max(Math.round(t._radiusY),1)||e)+" 0 1,0 ",o=t._empty()?"M0 0":"M"+(i.x-e)+","+i.y+n+2*e+",0 "+n+2*-e+",0 ";this._setPath(t,o)},_setPath:function(t,i){t._path.setAttribute("d",i)},_bringToFront:function(t){ct(t._path)},_bringToBack:function(t){_t(t._path)}});Ji&&xn.include(vn),Le.include({getRenderer:function(t){var i=t.options.renderer||this._getPaneRenderer(t.options.pane)||this.options.renderer||this._renderer;return i||(i=this._renderer=this.options.preferCanvas&&Xt()||Jt()),this.hasLayer(i)||this.addLayer(i),i},_getPaneRenderer:function(t){if("overlayPane"===t||void 0===t)return!1;var i=this._paneRenderers[t];return void 0===i&&(i=xn&&Jt({pane:t})||fn&&Xt({pane:t}),this._paneRenderers[t]=i),i}});var wn=en.extend({initialize:function(t,i){en.prototype.initialize.call(this,this._boundsToLatLngs(t),i)},setBounds:function(t){return this.setLatLngs(this._boundsToLatLngs(t))},_boundsToLatLngs:function(t){return t=z(t),[t.getSouthWest(),t.getNorthWest(),t.getNorthEast(),t.getSouthEast()]}});xn.create=yn,xn.pointsToPath=k,nn.geometryToLayer=Wt,nn.coordsToLatLng=Ht,nn.coordsToLatLngs=Ft,nn.latLngToCoords=Ut,nn.latLngsToCoords=Vt,nn.getFeature=qt,nn.asFeature=Gt,Le.mergeOptions({boxZoom:!0});var Ln=Ze.extend({initialize:function(t){this._map=t,this._container=t._container,this._pane=t._panes.overlayPane,this._resetStateTimeout=0,t.on("unload",this._destroy,this)},addHooks:function(){V(this._container,"mousedown",this._onMouseDown,this)},removeHooks:function(){q(this._container,"mousedown",this._onMouseDown,this)},moved:function(){return this._moved},_destroy:function(){ut(this._pane),delete this._pane},_resetState:function(){this._resetStateTimeout=0,this._moved=!1},_clearDeferredResetState:function(){0!==this._resetStateTimeout&&(clearTimeout(this._resetStateTimeout),this._resetStateTimeout=0)},_onMouseDown:function(t){if(!t.shiftKey||1!==t.which&&1!==t.button)return!1;this._clearDeferredResetState(),this._resetState(),mi(),bt(),this._startPoint=this._map.mouseEventToContainerPoint(t),V(document,{contextmenu:Q,mousemove:this._onMouseMove,mouseup:this._onMouseUp,keydown:this._onKeyDown},this)},_onMouseMove:function(t){this._moved||(this._moved=!0,this._box=ht("div","leaflet-zoom-box",this._container),pt(this._container,"leaflet-crosshair"),this._map.fire("boxzoomstart")),this._point=this._map.mouseEventToContainerPoint(t);var i=new P(this._point,this._startPoint),e=i.getSize();Lt(this._box,i.min),this._box.style.width=e.x+"px",this._box.style.height=e.y+"px"},_finish:function(){this._moved&&(ut(this._box),mt(this._container,"leaflet-crosshair")),fi(),Tt(),q(document,{contextmenu:Q,mousemove:this._onMouseMove,mouseup:this._onMouseUp,keydown:this._onKeyDown},this)},_onMouseUp:function(t){if((1===t.which||1===t.button)&&(this._finish(),this._moved)){this._clearDeferredResetState(),this._resetStateTimeout=setTimeout(e(this._resetState,this),0);var i=new T(this._map.containerPointToLatLng(this._startPoint),this._map.containerPointToLatLng(this._point));this._map.fitBounds(i).fire("boxzoomend",{boxZoomBounds:i})}},_onKeyDown:function(t){27===t.keyCode&&this._finish()}});Le.addInitHook("addHandler","boxZoom",Ln),Le.mergeOptions({doubleClickZoom:!0});var Pn=Ze.extend({addHooks:function(){this._map.on("dblclick",this._onDoubleClick,this)},removeHooks:function(){this._map.off("dblclick",this._onDoubleClick,this)},_onDoubleClick:function(t){var i=this._map,e=i.getZoom(),n=i.options.zoomDelta,o=t.originalEvent.shiftKey?e-n:e+n;"center"===i.options.doubleClickZoom?i.setZoom(o):i.setZoomAround(t.containerPoint,o)}});Le.addInitHook("addHandler","doubleClickZoom",Pn),Le.mergeOptions({dragging:!0,inertia:!zi,inertiaDeceleration:3400,inertiaMaxSpeed:1/0,easeLinearity:.2,worldCopyJump:!1,maxBoundsViscosity:0});var bn=Ze.extend({addHooks:function(){if(!this._draggable){var t=this._map;this._draggable=new Be(t._mapPane,t._container),this._draggable.on({dragstart:this._onDragStart,drag:this._onDrag,dragend:this._onDragEnd},this),this._draggable.on("predrag",this._onPreDragLimit,this),t.options.worldCopyJump&&(this._draggable.on("predrag",this._onPreDragWrap,this),t.on("zoomend",this._onZoomEnd,this),t.whenReady(this._onZoomEnd,this))}pt(this._map._container,"leaflet-grab leaflet-touch-drag"),this._draggable.enable(),this._positions=[],this._times=[]},removeHooks:function(){mt(this._map._container,"leaflet-grab"),mt(this._map._container,"leaflet-touch-drag"),this._draggable.disable()},moved:function(){return this._draggable&&this._draggable._moved},moving:function(){return this._draggable&&this._draggable._moving},_onDragStart:function(){var t=this._map;if(t._stop(),this._map.options.maxBounds&&this._map.options.maxBoundsViscosity){var i=z(this._map.options.maxBounds);this._offsetLimit=b(this._map.latLngToContainerPoint(i.getNorthWest()).multiplyBy(-1),this._map.latLngToContainerPoint(i.getSouthEast()).multiplyBy(-1).add(this._map.getSize())),this._viscosity=Math.min(1,Math.max(0,this._map.options.maxBoundsViscosity))}else this._offsetLimit=null;t.fire("movestart").fire("dragstart"),t.options.inertia&&(this._positions=[],this._times=[])},_onDrag:function(t){if(this._map.options.inertia){var i=this._lastTime=+new Date,e=this._lastPos=this._draggable._absPos||this._draggable._newPos;this._positions.push(e),this._times.push(i),this._prunePositions(i)}this._map.fire("move",t).fire("drag",t)},_prunePositions:function(t){for(;this._positions.length>1&&t-this._times[0]>50;)this._positions.shift(),this._times.shift()},_onZoomEnd:function(){var t=this._map.getSize().divideBy(2),i=this._map.latLngToLayerPoint([0,0]);this._initialWorldOffset=i.subtract(t).x,this._worldWidth=this._map.getPixelWorldBounds().getSize().x},_viscousLimit:function(t,i){return t-(t-i)*this._viscosity},_onPreDragLimit:function(){if(this._viscosity&&this._offsetLimit){var t=this._draggable._newPos.subtract(this._draggable._startPos),i=this._offsetLimit;t.x<i.min.x&&(t.x=this._viscousLimit(t.x,i.min.x)),t.y<i.min.y&&(t.y=this._viscousLimit(t.y,i.min.y)),t.x>i.max.x&&(t.x=this._viscousLimit(t.x,i.max.x)),t.y>i.max.y&&(t.y=this._viscousLimit(t.y,i.max.y)),this._draggable._newPos=this._draggable._startPos.add(t)}},_onPreDragWrap:function(){var t=this._worldWidth,i=Math.round(t/2),e=this._initialWorldOffset,n=this._draggable._newPos.x,o=(n-i+e)%t+i-e,s=(n+i+e)%t-i-e,r=Math.abs(o+e)<Math.abs(s+e)?o:s;this._draggable._absPos=this._draggable._newPos.clone(),this._draggable._newPos.x=r},_onDragEnd:function(t){var i=this._map,e=i.options,n=!e.inertia||this._times.length<2;if(i.fire("dragend",t),n)i.fire("moveend");else{this._prunePositions(+new Date);var o=this._lastPos.subtract(this._positions[0]),s=(this._lastTime-this._times[0])/1e3,r=e.easeLinearity,a=o.multiplyBy(r/s),h=a.distanceTo([0,0]),u=Math.min(e.inertiaMaxSpeed,h),l=a.multiplyBy(u/h),c=u/(e.inertiaDeceleration*r),_=l.multiplyBy(-c/2).round();_.x||_.y?(_=i._limitOffset(_,i.options.maxBounds),f(function(){i.panBy(_,{duration:c,easeLinearity:r,noMoveStart:!0,animate:!0})})):i.fire("moveend")}}});Le.addInitHook("addHandler","dragging",bn),Le.mergeOptions({keyboard:!0,keyboardPanDelta:80});var Tn=Ze.extend({keyCodes:{left:[37],right:[39],down:[40],up:[38],zoomIn:[187,107,61,171],zoomOut:[189,109,54,173]},initialize:function(t){this._map=t,this._setPanDelta(t.options.keyboardPanDelta),this._setZoomDelta(t.options.zoomDelta)},addHooks:function(){var t=this._map._container;t.tabIndex<=0&&(t.tabIndex="0"),V(t,{focus:this._onFocus,blur:this._onBlur,mousedown:this._onMouseDown},this),this._map.on({focus:this._addHooks,blur:this._removeHooks},this)},removeHooks:function(){this._removeHooks(),q(this._map._container,{focus:this._onFocus,blur:this._onBlur,mousedown:this._onMouseDown},this),this._map.off({focus:this._addHooks,blur:this._removeHooks},this)},_onMouseDown:function(){if(!this._focused){var t=document.body,i=document.documentElement,e=t.scrollTop||i.scrollTop,n=t.scrollLeft||i.scrollLeft;this._map._container.focus(),window.scrollTo(n,e)}},_onFocus:function(){this._focused=!0,this._map.fire("focus")},_onBlur:function(){this._focused=!1,this._map.fire("blur")},_setPanDelta:function(t){var i,e,n=this._panKeys={},o=this.keyCodes;for(i=0,e=o.left.length;i<e;i++)n[o.left[i]]=[-1*t,0];for(i=0,e=o.right.length;i<e;i++)n[o.right[i]]=[t,0];for(i=0,e=o.down.length;i<e;i++)n[o.down[i]]=[0,t];for(i=0,e=o.up.length;i<e;i++)n[o.up[i]]=[0,-1*t]},_setZoomDelta:function(t){var i,e,n=this._zoomKeys={},o=this.keyCodes;for(i=0,e=o.zoomIn.length;i<e;i++)n[o.zoomIn[i]]=t;for(i=0,e=o.zoomOut.length;i<e;i++)n[o.zoomOut[i]]=-t},_addHooks:function(){V(document,"keydown",this._onKeyDown,this)},_removeHooks:function(){q(document,"keydown",this._onKeyDown,this)},_onKeyDown:function(t){if(!(t.altKey||t.ctrlKey||t.metaKey)){var i,e=t.keyCode,n=this._map;if(e in this._panKeys){if(n._panAnim&&n._panAnim._inProgress)return;i=this._panKeys[e],t.shiftKey&&(i=w(i).multiplyBy(3)),n.panBy(i),n.options.maxBounds&&n.panInsideBounds(n.options.maxBounds)}else if(e in this._zoomKeys)n.setZoom(n.getZoom()+(t.shiftKey?3:1)*this._zoomKeys[e]);else{if(27!==e||!n._popup||!n._popup.options.closeOnEscapeKey)return;n.closePopup()}Q(t)}}});Le.addInitHook("addHandler","keyboard",Tn),Le.mergeOptions({scrollWheelZoom:!0,wheelDebounceTime:40,wheelPxPerZoomLevel:60});var zn=Ze.extend({addHooks:function(){V(this._map._container,"mousewheel",this._onWheelScroll,this),this._delta=0},removeHooks:function(){q(this._map._container,"mousewheel",this._onWheelScroll,this)},_onWheelScroll:function(t){var i=it(t),n=this._map.options.wheelDebounceTime;this._delta+=i,this._lastMousePos=this._map.mouseEventToContainerPoint(t),this._startTime||(this._startTime=+new Date);var o=Math.max(n-(+new Date-this._startTime),0);clearTimeout(this._timer),this._timer=setTimeout(e(this._performZoom,this),o),Q(t)},_performZoom:function(){var t=this._map,i=t.getZoom(),e=this._map.options.zoomSnap||0;t._stop();var n=this._delta/(4*this._map.options.wheelPxPerZoomLevel),o=4*Math.log(2/(1+Math.exp(-Math.abs(n))))/Math.LN2,s=e?Math.ceil(o/e)*e:o,r=t._limitZoom(i+(this._delta>0?s:-s))-i;this._delta=0,this._startTime=null,r&&("center"===t.options.scrollWheelZoom?t.setZoom(i+r):t.setZoomAround(this._lastMousePos,i+r))}});Le.addInitHook("addHandler","scrollWheelZoom",zn),Le.mergeOptions({tap:!0,tapTolerance:15});var Mn=Ze.extend({addHooks:function(){V(this._map._container,"touchstart",this._onDown,this)},removeHooks:function(){q(this._map._container,"touchstart",this._onDown,this)},_onDown:function(t){if(t.touches){if($(t),this._fireClick=!0,t.touches.length>1)return this._fireClick=!1,void clearTimeout(this._holdTimeout);var i=t.touches[0],n=i.target;this._startPos=this._newPos=new x(i.clientX,i.clientY),n.tagName&&"a"===n.tagName.toLowerCase()&&pt(n,"leaflet-active"),this._holdTimeout=setTimeout(e(function(){this._isTapValid()&&(this._fireClick=!1,this._onUp(),this._simulateEvent("contextmenu",i))},this),1e3),this._simulateEvent("mousedown",i),V(document,{touchmove:this._onMove,touchend:this._onUp},this)}},_onUp:function(t){if(clearTimeout(this._holdTimeout),q(document,{touchmove:this._onMove,touchend:this._onUp},this),this._fireClick&&t&&t.changedTouches){var i=t.changedTouches[0],e=i.target;e&&e.tagName&&"a"===e.tagName.toLowerCase()&&mt(e,"leaflet-active"),this._simulateEvent("mouseup",i),this._isTapValid()&&this._simulateEvent("click",i)}},_isTapValid:function(){return this._newPos.distanceTo(this._startPos)<=this._map.options.tapTolerance},_onMove:function(t){var i=t.touches[0];this._newPos=new x(i.clientX,i.clientY),this._simulateEvent("mousemove",i)},_simulateEvent:function(t,i){var e=document.createEvent("MouseEvents");e._simulated=!0,i.target._simulatedClick=!0,e.initMouseEvent(t,!0,!0,window,1,i.screenX,i.screenY,i.clientX,i.clientY,!1,!1,!1,!1,0,null),i.target.dispatchEvent(e)}});Vi&&!Ui&&Le.addInitHook("addHandler","tap",Mn),Le.mergeOptions({touchZoom:Vi&&!zi,bounceAtZoomLimits:!0});var Cn=Ze.extend({addHooks:function(){pt(this._map._container,"leaflet-touch-zoom"),V(this._map._container,"touchstart",this._onTouchStart,this)},removeHooks:function(){mt(this._map._container,"leaflet-touch-zoom"),q(this._map._container,"touchstart",this._onTouchStart,this)},_onTouchStart:function(t){var i=this._map;if(t.touches&&2===t.touches.length&&!i._animatingZoom&&!this._zooming){var e=i.mouseEventToContainerPoint(t.touches[0]),n=i.mouseEventToContainerPoint(t.touches[1]);this._centerPoint=i.getSize()._divideBy(2),this._startLatLng=i.containerPointToLatLng(this._centerPoint),"center"!==i.options.touchZoom&&(this._pinchStartLatLng=i.containerPointToLatLng(e.add(n)._divideBy(2))),this._startDist=e.distanceTo(n),this._startZoom=i.getZoom(),this._moved=!1,this._zooming=!0,i._stop(),V(document,"touchmove",this._onTouchMove,this),V(document,"touchend",this._onTouchEnd,this),$(t)}},_onTouchMove:function(t){if(t.touches&&2===t.touches.length&&this._zooming){var i=this._map,n=i.mouseEventToContainerPoint(t.touches[0]),o=i.mouseEventToContainerPoint(t.touches[1]),s=n.distanceTo(o)/this._startDist;if(this._zoom=i.getScaleZoom(s,this._startZoom),!i.options.bounceAtZoomLimits&&(this._zoom<i.getMinZoom()&&s<1||this._zoom>i.getMaxZoom()&&s>1)&&(this._zoom=i._limitZoom(this._zoom)),"center"===i.options.touchZoom){if(this._center=this._startLatLng,1===s)return}else{var r=n._add(o)._divideBy(2)._subtract(this._centerPoint);if(1===s&&0===r.x&&0===r.y)return;this._center=i.unproject(i.project(this._pinchStartLatLng,this._zoom).subtract(r),this._zoom)}this._moved||(i._moveStart(!0,!1),this._moved=!0),g(this._animRequest);var a=e(i._move,i,this._center,this._zoom,{pinch:!0,round:!1});this._animRequest=f(a,this,!0),$(t)}},_onTouchEnd:function(){this._moved&&this._zooming?(this._zooming=!1,g(this._animRequest),q(document,"touchmove",this._onTouchMove),q(document,"touchend",this._onTouchEnd),this._map.options.zoomAnimation?this._map._animateZoom(this._center,this._map._limitZoom(this._zoom),!0,this._map.options.zoomSnap):this._map._resetView(this._center,this._map._limitZoom(this._zoom))):this._zooming=!1}});Le.addInitHook("addHandler","touchZoom",Cn),Le.BoxZoom=Ln,Le.DoubleClickZoom=Pn,Le.Drag=bn,Le.Keyboard=Tn,Le.ScrollWheelZoom=zn,Le.Tap=Mn,Le.TouchZoom=Cn;var Zn=window.L;window.L=t,Object.freeze=$t,t.version="1.3.1+HEAD.ba6f97f",t.noConflict=function(){return window.L=Zn,this},t.Control=Pe,t.control=be,t.Browser=$i,t.Evented=ui,t.Mixin=Ee,t.Util=ai,t.Class=v,t.Handler=Ze,t.extend=i,t.bind=e,t.stamp=n,t.setOptions=l,t.DomEvent=de,t.DomUtil=xe,t.PosAnimation=we,t.Draggable=Be,t.LineUtil=Oe,t.PolyUtil=Re,t.Point=x,t.point=w,t.Bounds=P,t.bounds=b,t.Transformation=Z,t.transformation=S,t.Projection=je,t.LatLng=M,t.latLng=C,t.LatLngBounds=T,t.latLngBounds=z,t.CRS=ci,t.GeoJSON=nn,t.geoJSON=Kt,t.geoJson=sn,t.Layer=Ue,t.LayerGroup=Ve,t.layerGroup=function(t,i){return new Ve(t,i)},t.FeatureGroup=qe,t.featureGroup=function(t){return new qe(t)},t.ImageOverlay=rn,t.imageOverlay=function(t,i,e){return new rn(t,i,e)},t.VideoOverlay=an,t.videoOverlay=function(t,i,e){return new an(t,i,e)},t.DivOverlay=hn,t.Popup=un,t.popup=function(t,i){return new un(t,i)},t.Tooltip=ln,t.tooltip=function(t,i){return new ln(t,i)},t.Icon=Ge,t.icon=function(t){return new Ge(t)},t.DivIcon=cn,t.divIcon=function(t){return new cn(t)},t.Marker=Xe,t.marker=function(t,i){return new Xe(t,i)},t.TileLayer=dn,t.tileLayer=Yt,t.GridLayer=_n,t.gridLayer=function(t){return new _n(t)},t.SVG=xn,t.svg=Jt,t.Renderer=mn,t.Canvas=fn,t.canvas=Xt,t.Path=Je,t.CircleMarker=$e,t.circleMarker=function(t,i){return new $e(t,i)},t.Circle=Qe,t.circle=function(t,i,e){return new Qe(t,i,e)},t.Polyline=tn,t.polyline=function(t,i){return new tn(t,i)},t.Polygon=en,t.polygon=function(t,i){return new en(t,i)},t.Rectangle=wn,t.rectangle=function(t,i){return new wn(t,i)},t.Map=Le,t.map=function(t,i){return new Le(t,i)}});</script>
<style type="text/css">
img.leaflet-tile {
padding: 0;
margin: 0;
border-radius: 0;
border: none;
}
.info {
padding: 6px 8px;
font: 14px/16px Arial, Helvetica, sans-serif;
background: white;
background: rgba(255,255,255,0.8);
box-shadow: 0 0 15px rgba(0,0,0,0.2);
border-radius: 5px;
}
.legend {
line-height: 18px;
color: #555;
}
.legend svg text {
fill: #555;
}
.legend svg line {
stroke: #555;
}
.legend i {
width: 18px;
height: 18px;
margin-right: 4px;
opacity: 0.7;
display: inline-block;
vertical-align: top;

zoom: 1;
*display: inline;
}
</style>
<script>!function(t,s){"object"==typeof exports&&"undefined"!=typeof module?module.exports=s():"function"==typeof define&&define.amd?define(s):t.proj4=s()}(this,function(){"use strict";function k(t,s){if(t[s])return t[s];for(var i,a=Object.keys(t),h=s.toLowerCase().replace(H,""),e=-1;++e<a.length;)if((i=a[e]).toLowerCase().replace(H,"")===h)return t[i]}function e(t){if("string"!=typeof t)throw new Error("not a string");this.text=t.trim(),this.level=0,this.place=0,this.root=null,this.stack=[],this.currentObject=null,this.state=K}function h(t,s,i){Array.isArray(s)&&(i.unshift(s),s=null);var a=s?{}:t,h=i.reduce(function(t,s){return n(s,t),t},a);s&&(t[s]=h)}function n(t,s){if(Array.isArray(t)){var i,a=t.shift();if("PARAMETER"===a&&(a=t.shift()),1===t.length)return Array.isArray(t[0])?(s[a]={},void n(t[0],s[a])):void(s[a]=t[0]);if(t.length)if("TOWGS84"!==a){if("AXIS"===a)return a in s||(s[a]=[]),void s[a].push(t);switch(Array.isArray(a)||(s[a]={}),a){case"UNIT":case"PRIMEM":case"VERT_DATUM":return s[a]={name:t[0].toLowerCase(),convert:t[1]},void(3===t.length&&n(t[2],s[a]));case"SPHEROID":case"ELLIPSOID":return s[a]={name:t[0],a:t[1],rf:t[2]},void(4===t.length&&n(t[3],s[a]));case"PROJECTEDCRS":case"PROJCRS":case"GEOGCS":case"GEOCCS":case"PROJCS":case"LOCAL_CS":case"GEODCRS":case"GEODETICCRS":case"GEODETICDATUM":case"EDATUM":case"ENGINEERINGDATUM":case"VERT_CS":case"VERTCRS":case"VERTICALCRS":case"COMPD_CS":case"COMPOUNDCRS":case"ENGINEERINGCRS":case"ENGCRS":case"FITTED_CS":case"LOCAL_DATUM":case"DATUM":return t[0]=["name",t[0]],void h(s,a,t);default:for(i=-1;++i<t.length;)if(!Array.isArray(t[i]))return n(t,s[a]);return h(s,a,t)}}else s[a]=t;else s[a]=!0}else s[t]=!0}function r(t){return t*it}function o(e){function t(t){return t*(e.to_meter||1)}if("GEOGCS"===e.type?e.projName="longlat":"LOCAL_CS"===e.type?(e.projName="identity",e.local=!0):"object"==typeof e.PROJECTION?e.projName=Object.keys(e.PROJECTION)[0]:e.projName=e.PROJECTION,e.AXIS){for(var s="",i=0,a=e.AXIS.length;i<a;++i){var h=e.AXIS[i][0].toLowerCase();-1!==h.indexOf("north")?s+="n":-1!==h.indexOf("south")?s+="s":-1!==h.indexOf("east")?s+="e":-1!==h.indexOf("west")&&(s+="w")}2===s.length&&(s+="u"),3===s.length&&(e.axis=s)}e.UNIT&&(e.units=e.UNIT.name.toLowerCase(),"metre"===e.units&&(e.units="meter"),e.UNIT.convert&&("GEOGCS"===e.type?e.DATUM&&e.DATUM.SPHEROID&&(e.to_meter=e.UNIT.convert*e.DATUM.SPHEROID.a):e.to_meter=e.UNIT.convert));var n=e.GEOGCS;"GEOGCS"===e.type&&(n=e),n&&(n.DATUM?e.datumCode=n.DATUM.name.toLowerCase():e.datumCode=n.name.toLowerCase(),"d_"===e.datumCode.slice(0,2)&&(e.datumCode=e.datumCode.slice(2)),"new_zealand_geodetic_datum_1949"!==e.datumCode&&"new_zealand_1949"!==e.datumCode||(e.datumCode="nzgd49"),"wgs_1984"!==e.datumCode&&"world_geodetic_system_1984"!==e.datumCode||("Mercator_Auxiliary_Sphere"===e.PROJECTION&&(e.sphere=!0),e.datumCode="wgs84"),"_ferro"===e.datumCode.slice(-6)&&(e.datumCode=e.datumCode.slice(0,-6)),"_jakarta"===e.datumCode.slice(-8)&&(e.datumCode=e.datumCode.slice(0,-8)),~e.datumCode.indexOf("belge")&&(e.datumCode="rnb72"),n.DATUM&&n.DATUM.SPHEROID&&(e.ellps=n.DATUM.SPHEROID.name.replace("_19","").replace(/[Cc]larke\_18/,"clrk"),"international"===e.ellps.toLowerCase().slice(0,13)&&(e.ellps="intl"),e.a=n.DATUM.SPHEROID.a,e.rf=parseFloat(n.DATUM.SPHEROID.rf,10)),n.DATUM&&n.DATUM.TOWGS84&&(e.datum_params=n.DATUM.TOWGS84),~e.datumCode.indexOf("osgb_1936")&&(e.datumCode="osgb36"),~e.datumCode.indexOf("osni_1952")&&(e.datumCode="osni52"),(~e.datumCode.indexOf("tm65")||~e.datumCode.indexOf("geodetic_datum_of_1965"))&&(e.datumCode="ire65"),"ch1903+"===e.datumCode&&(e.datumCode="ch1903"),~e.datumCode.indexOf("israel")&&(e.datumCode="isr93")),e.b&&!isFinite(e.b)&&(e.b=e.a),[["standard_parallel_1","Standard_Parallel_1"],["standard_parallel_2","Standard_Parallel_2"],["false_easting","False_Easting"],["false_northing","False_Northing"],["central_meridian","Central_Meridian"],["latitude_of_origin","Latitude_Of_Origin"],["latitude_of_origin","Central_Parallel"],["scale_factor","Scale_Factor"],["k0","scale_factor"],["latitude_of_center","Latitude_Of_Center"],["latitude_of_center","Latitude_of_center"],["lat0","latitude_of_center",r],["longitude_of_center","Longitude_Of_Center"],["longitude_of_center","Longitude_of_center"],["longc","longitude_of_center",r],["x0","false_easting",t],["y0","false_northing",t],["long0","central_meridian",r],["lat0","latitude_of_origin",r],["lat0","standard_parallel_1",r],["lat1","standard_parallel_1",r],["lat2","standard_parallel_2",r],["azimuth","Azimuth"],["alpha","azimuth",r],["srsCode","name"]].forEach(function(t){return s=e,a=(i=t)[0],h=i[1],void(!(a in s)&&h in s&&(s[a]=s[h],3===i.length&&(s[a]=i[2](s[a]))));var s,i,a,h}),e.long0||!e.longc||"Albers_Conic_Equal_Area"!==e.projName&&"Lambert_Azimuthal_Equal_Area"!==e.projName||(e.long0=e.longc),e.lat_ts||!e.lat1||"Stereographic_South_Pole"!==e.projName&&"Polar Stereographic (variant B)"!==e.projName||(e.lat0=r(0<e.lat1?90:-90),e.lat_ts=e.lat1)}function l(t){var s=this;if(2===arguments.length){var i=arguments[1];"string"==typeof i?"+"===i.charAt(0)?l[t]=J(arguments[1]):l[t]=at(arguments[1]):l[t]=i}else if(1===arguments.length){if(Array.isArray(t))return t.map(function(t){Array.isArray(t)?l.apply(s,t):l(t)});if("string"==typeof t){if(t in l)return l[t]}else"EPSG"in t?l["EPSG:"+t.EPSG]=t:"ESRI"in t?l["ESRI:"+t.ESRI]=t:"IAU2000"in t?l["IAU2000:"+t.IAU2000]=t:console.log(t);return}}function E(t){if("string"!=typeof t)return t;if(t in l)return l[t];if(a=t,lt.some(function(t){return-1<a.indexOf(t)})){var s=at(t);if(function(t){var s=k(t,"authority");if(s){var i=k(s,"epsg");return i&&-1<Mt.indexOf(i)}}(s))return l["EPSG:3857"];var i=function(t){var s=k(t,"extension");if(s)return k(s,"proj4")}(s);return i?J(i):s}var a;return"+"===t[0]?J(t):void 0}function t(t){return t}function s(t,s){var i=mt.length;return t.names?((mt[i]=t).names.forEach(function(t){ft[t.toLowerCase()]=i}),this):(console.log(s),!0)}function q(t,s){if(!(this instanceof q))return new q(t);s=s||function(t){if(t)throw t};var i,a,h,e,n,r,o,l,M,c,u,f,m,p,d,y,_,x,g,b,v,w,C,P,S,N=E(t);"object"==typeof N&&(i=q.projections.get(N.projName))?(!N.datumCode||"none"===N.datumCode||(a=k(_t,N.datumCode))&&(N.datum_params=a.towgs84?a.towgs84.split(","):null,N.ellps=a.ellipse,N.datumName=a.datumName?a.datumName:N.datumCode),N.k0=N.k0||1,N.axis=N.axis||"enu",N.ellps=N.ellps||"wgs84",b=N.a,v=N.b,w=N.rf,C=N.ellps,P=N.sphere,b||(b=(S=(S=k(dt,C))||yt).a,v=S.b,w=S.rf),w&&!v&&(v=(1-1/w)*b),(0===w||Math.abs(b-v)<D)&&(P=!0,v=b),m=(h={a:b,b:v,rf:w,sphere:P}).a,p=h.b,d=N.R_A,x=((y=m*m)-(_=p*p))/y,g=0,d?(y=(m*=1-x*(R+x*(L+x*T)))*m,x=0):g=Math.sqrt(x),e={es:x,e:g,ep2:(y-_)/_},n=N.datum||(r=N.datumCode,o=N.datum_params,l=h.a,M=h.b,c=e.es,u=e.ep2,(f={}).datum_type=void 0===r||"none"===r?G:A,o&&(f.datum_params=o.map(parseFloat),0===f.datum_params[0]&&0===f.datum_params[1]&&0===f.datum_params[2]||(f.datum_type=I),3<f.datum_params.length&&(0===f.datum_params[3]&&0===f.datum_params[4]&&0===f.datum_params[5]&&0===f.datum_params[6]||(f.datum_type=O,f.datum_params[3]*=j,f.datum_params[4]*=j,f.datum_params[5]*=j,f.datum_params[6]=f.datum_params[6]/1e6+1))),f.a=l,f.b=M,f.es=c,f.ep2=u,f),ct(this,N),ct(this,i),this.a=h.a,this.b=h.b,this.rf=h.rf,this.sphere=h.sphere,this.es=e.es,this.e=e.e,this.ep2=e.ep2,this.datum=n,this.init(),s(null,this)):s(t)}function M(t,s,i){var a,h,e,n,r=t.x,o=t.y,l=t.z?t.z:0;if(o<-z&&-1.001*z<o)o=-z;else if(z<o&&o<1.001*z)o=z;else{if(o<-z)return{x:-1/0,y:-1/0,z:t.z};if(z<o)return{x:1/0,y:1/0,z:t.z}}return r>Math.PI&&(r-=2*Math.PI),h=Math.sin(o),n=Math.cos(o),e=h*h,{x:((a=i/Math.sqrt(1-s*e))+l)*n*Math.cos(r),y:(a+l)*n*Math.sin(r),z:(a*(1-s)+l)*h}}function c(t,s,i,a){var h,e,n,r,o,l,M,c,u,f,m,p,d,y=t.x,_=t.y,x=t.z?t.z:0,g=Math.sqrt(y*y+_*_),b=Math.sqrt(y*y+_*_+x*x);if(g/i<1e-12){if(p=0,b/i<1e-12)return d=-a,{x:t.x,y:t.y,z:t.z}}else p=Math.atan2(_,y);for(h=x/b,l=(e=g/b)*(1-s)*(n=1/Math.sqrt(1-s*(2-s)*e*e)),M=h*n,m=0;m++,r=s*(o=i/Math.sqrt(1-s*M*M))/(o+(d=g*l+x*M-o*(1-s*M*M))),f=(u=h*(n=1/Math.sqrt(1-r*(2-r)*e*e)))*l-(c=e*(1-r)*n)*M,l=c,M=u,1e-24<f*f&&m<30;);return{x:p,y:Math.atan(u/Math.abs(c)),z:d}}function u(t){return t===I||t===O}function i(t){if("function"==typeof Number.isFinite){if(Number.isFinite(t))return;throw new TypeError("coordinates must be finite numbers")}if("number"!=typeof t||t!=t||!isFinite(t))throw new TypeError("coordinates must be finite numbers")}function f(t,s,i){var a,h,e;if(Array.isArray(i)&&(i=bt(i)),vt(i),t.datum&&s.datum&&(e=s,((h=t).datum.datum_type===I||h.datum.datum_type===O)&&"WGS84"!==e.datumCode||(e.datum.datum_type===I||e.datum.datum_type===O)&&"WGS84"!==h.datumCode)&&(i=f(t,a=new q("WGS84"),i),t=a),"enu"!==t.axis&&(i=gt(t,!1,i)),"longlat"===t.projName)i={x:i.x*N,y:i.y*N,z:i.z||0};else if(t.to_meter&&(i={x:i.x*t.to_meter,y:i.y*t.to_meter,z:i.z||0}),!(i=t.inverse(i)))return;return t.from_greenwich&&(i.x+=t.from_greenwich),i=xt(t.datum,s.datum,i),s.from_greenwich&&(i={x:i.x-s.from_greenwich,y:i.y,z:i.z||0}),"longlat"===s.projName?i={x:i.x*B,y:i.y*B,z:i.z||0}:(i=s.forward(i),s.to_meter&&(i={x:i.x/s.to_meter,y:i.y/s.to_meter,z:i.z||0})),"enu"!==s.axis?gt(s,!0,i):i}function m(s,i,a){var t,h,e;return Array.isArray(a)?(t=f(s,i,a)||{x:NaN,y:NaN},2<a.length?void 0!==s.name&&"geocent"===s.name||void 0!==i.name&&"geocent"===i.name?"number"==typeof t.z?[t.x,t.y,t.z].concat(a.splice(3)):[t.x,t.y,a[2]].concat(a.splice(3)):[t.x,t.y].concat(a.splice(2)):[t.x,t.y]):(h=f(s,i,a),2===(e=Object.keys(a)).length||e.forEach(function(t){if(void 0!==s.name&&"geocent"===s.name||void 0!==i.name&&"geocent"===i.name){if("x"===t||"y"===t||"z"===t)return}else if("x"===t||"y"===t)return;h[t]=a[t]}),h)}function p(t){return t instanceof q?t:t.oProj?t.oProj:q(t)}function a(s,i,t){s=p(s);var a,h=!1;return void 0===i?(i=s,s=wt,h=!0):void 0===i.x&&!Array.isArray(i)||(t=i,i=s,s=wt,h=!0),i=p(i),t?m(s,i,t):(a={forward:function(t){return m(s,i,t)},inverse:function(t){return m(i,s,t)}},h&&(a.oProj=i),a)}function d(t,s){return s=s||5,i=function(t){var s,i,a,h,e,n,r=t.lat,o=t.lon,l=_(r),M=_(o);n=Math.floor((o+180)/6)+1,180===o&&(n=60),56<=r&&r<64&&3<=o&&o<12&&(n=32),72<=r&&r<84&&(0<=o&&o<9?n=31:9<=o&&o<21?n=33:21<=o&&o<33?n=35:33<=o&&o<42&&(n=37)),e=_(6*(n-1)-180+3),s=6378137/Math.sqrt(1-.00669438*Math.sin(l)*Math.sin(l)),i=Math.tan(l)*Math.tan(l),a=.006739496752268451*Math.cos(l)*Math.cos(l);var c=.9996*s*((h=Math.cos(l)*(M-e))+(1-i+a)*h*h*h/6+(5-18*i+i*i+72*a-.39089081163157013)*h*h*h*h*h/120)+5e5,u=.9996*(6378137*(.9983242984503243*l-.002514607064228144*Math.sin(2*l)+2639046602129982e-21*Math.sin(4*l)-3.418046101696858e-9*Math.sin(6*l))+s*Math.tan(l)*(h*h/2+(5-i+9*a+4*a*a)*h*h*h*h/24+(61-58*i+i*i+600*a-2.2240339282485886)*h*h*h*h*h*h/720));return r<0&&(u+=1e7),{northing:Math.round(u),easting:Math.round(c),zoneNumber:n,zoneLetter:function(t){var s="Z";return t<=84&&72<=t?s="X":t<72&&64<=t?s="W":t<64&&56<=t?s="V":t<56&&48<=t?s="U":t<48&&40<=t?s="T":t<40&&32<=t?s="S":t<32&&24<=t?s="R":t<24&&16<=t?s="Q":t<16&&8<=t?s="P":t<8&&0<=t?s="N":t<0&&-8<=t?s="M":t<-8&&-16<=t?s="L":t<-16&&-24<=t?s="K":t<-24&&-32<=t?s="J":t<-32&&-40<=t?s="H":t<-40&&-48<=t?s="G":t<-48&&-56<=t?s="F":t<-56&&-64<=t?s="E":t<-64&&-72<=t?s="D":t<-72&&-80<=t&&(s="C"),s}(r)}}({lat:t[1],lon:t[0]}),a=s,h="00000"+i.easting,e="00000"+i.northing,i.zoneNumber+i.zoneLetter+function(t,s,i){var a=b(i);return function(t,s,i){var a=i-1,h=Pt.charCodeAt(a),e=St.charCodeAt(a),n=h+t-1,r=e+s,o=!1;return It<n&&(n=n-It+Nt-1,o=!0),(n===kt||h<kt&&kt<n||(kt<n||h<kt)&&o)&&n++,(n===Et||h<Et&&Et<n||(Et<n||h<Et)&&o)&&++n===kt&&n++,It<n&&(n=n-It+Nt-1),o=qt<r&&(r=r-qt+Nt-1,!0),(r===kt||e<kt&&kt<r||(kt<r||e<kt)&&o)&&r++,(r===Et||e<Et&&Et<r||(Et<r||e<Et)&&o)&&++r===kt&&r++,qt<r&&(r=r-qt+Nt-1),String.fromCharCode(n)+String.fromCharCode(r)}(Math.floor(t/1e5),Math.floor(s/1e5)%20,a)}(i.easting,i.northing,i.zoneNumber)+h.substr(h.length-5,a)+e.substr(e.length-5,a);var i,a,h,e}function y(t){var s=g(v(t.toUpperCase()));return s.lat&&s.lon?[s.lon,s.lat]:[(s.left+s.right)/2,(s.top+s.bottom)/2]}function _(t){return t*(Math.PI/180)}function x(t){return t/Math.PI*180}function g(t){var s=t.northing,i=t.easting,a=t.zoneLetter,h=t.zoneNumber;if(h<0||60<h)return null;var e,n,r,o,l,M,c,u,f=(1-Math.sqrt(.99330562))/(1+Math.sqrt(.99330562)),m=i-5e5,p=s;a<"N"&&(p-=1e7),M=6*(h-1)-180+3,u=(c=p/.9996/6367449.145945056)+(3*f/2-27*f*f*f/32)*Math.sin(2*c)+(21*f*f/16-55*f*f*f*f/32)*Math.sin(4*c)+151*f*f*f/96*Math.sin(6*c),e=6378137/Math.sqrt(1-.00669438*Math.sin(u)*Math.sin(u)),n=Math.tan(u)*Math.tan(u),r=.006739496752268451*Math.cos(u)*Math.cos(u),o=6335439.32722994/Math.pow(1-.00669438*Math.sin(u)*Math.sin(u),1.5),l=m/(.9996*e);var d,y=x(y=u-e*Math.tan(u)/o*(l*l/2-(5+3*n+10*r-4*r*r-.06065547077041606)*l*l*l*l/24+(61+90*n+298*r+45*n*n-1.6983531815716497-3*r*r)*l*l*l*l*l*l/720)),_=M+x(_=(l-(1+2*n+r)*l*l*l/6+(5-2*r+28*n-3*r*r+.05391597401814761+24*n*n)*l*l*l*l*l/120)/Math.cos(u));return t.accuracy?{top:(d=g({northing:t.northing+t.accuracy,easting:t.easting+t.accuracy,zoneLetter:t.zoneLetter,zoneNumber:t.zoneNumber})).lat,right:d.lon,bottom:y,left:_}:{lat:y,lon:_}}function b(t){var s=t%Ct;return 0===s&&(s=Ct),s}function v(t){if(t&&0===t.length)throw"MGRSPoint coverting from nothing";for(var s,i=t.length,a=null,h="",e=0;!/[A-Z]/.test(s=t.charAt(e));){if(2<=e)throw"MGRSPoint bad conversion from: "+t;h+=s,e++}var n=parseInt(h,10);if(0===e||i<e+3)throw"MGRSPoint bad conversion from: "+t;var r=t.charAt(e++);if(r<="A"||"B"===r||"Y"===r||"Z"<=r||"I"===r||"O"===r)throw"MGRSPoint zone letter "+r+" not handled: "+t;a=t.substring(e,e+=2);for(var o=b(n),l=function(t,s){for(var i=Pt.charCodeAt(s-1),a=1e5,h=!1;i!==t.charCodeAt(0);){if(++i===kt&&i++,i===Et&&i++,It<i){if(h)throw"Bad character: "+t;i=Nt,h=!0}a+=1e5}return a}(a.charAt(0),o),M=function(t,s){if("V"<t)throw"MGRSPoint given invalid Northing "+t;for(var i=St.charCodeAt(s-1),a=0,h=!1;i!==t.charCodeAt(0);){if(++i===kt&&i++,i===Et&&i++,qt<i){if(h)throw"Bad character: "+t;i=Nt,h=!0}a+=1e5}return a}(a.charAt(1),o);M<w(r);)M+=2e6;var c=i-e;if(c%2!=0)throw"MGRSPoint has to have an even number \nof digits after the zone letter and two 100km letters - front \nhalf for easting meters, second half for \nnorthing meters"+t;var u,f,m,p=c/2,d=0,y=0;return 0<p&&(u=1e5/Math.pow(10,p),f=t.substring(e,e+p),d=parseFloat(f)*u,m=t.substring(e+p),y=parseFloat(m)*u),{easting:d+l,northing:y+M,zoneLetter:r,zoneNumber:n,accuracy:u}}function w(t){var s;switch(t){case"C":s=11e5;break;case"D":s=2e6;break;case"E":s=28e5;break;case"F":s=37e5;break;case"G":s=46e5;break;case"H":s=55e5;break;case"J":s=64e5;break;case"K":s=73e5;break;case"L":s=82e5;break;case"M":s=91e5;break;case"N":s=0;break;case"P":s=8e5;break;case"Q":s=17e5;break;case"R":s=26e5;break;case"S":s=35e5;break;case"T":s=44e5;break;case"U":s=53e5;break;case"V":s=62e5;break;case"W":s=7e6;break;case"X":s=79e5;break;default:s=-1}if(0<=s)return s;throw"Invalid zone letter: "+t}function C(t,s,i){if(!(this instanceof C))return new C(t,s,i);var a;Array.isArray(t)?(this.x=t[0],this.y=t[1],this.z=t[2]||0):"object"==typeof t?(this.x=t.x,this.y=t.y,this.z=t.z||0):"string"==typeof t&&void 0===s?(a=t.split(","),this.x=parseFloat(a[0],10),this.y=parseFloat(a[1],10),this.z=parseFloat(a[2],10)||0):(this.x=t,this.y=s,this.z=i||0),console.warn("proj4.Point will be removed in version 3, use proj4.toPoint")}function P(t,s,i,a){var h;return t<D?(a.value=Os,h=0):(h=Math.atan2(s,i),Math.abs(h)<=U?a.value=Os:U<h&&h<=z+U?(a.value=As,h-=z):z+U<h||h<=-(z+U)?(a.value=Gs,h=0<=h?h-Q:h+Q):(a.value=js,h+=z)),h}function S(t,s){var i=t+s;return i<-Q?i+=F:+Q<i&&(i-=F),i}var I=1,O=2,A=4,G=5,j=484813681109536e-20,z=Math.PI/2,R=.16666666666666666,L=.04722222222222222,T=.022156084656084655,D=1e-10,N=.017453292519943295,B=57.29577951308232,U=Math.PI/4,F=2*Math.PI,Q=3.14159265359,W={greenwich:0,lisbon:-9.131906111111,paris:2.337229166667,bogota:-74.080916666667,madrid:-3.687938888889,rome:12.452333333333,bern:7.439583333333,jakarta:106.807719444444,ferro:-17.666666666667,brussels:4.367975,stockholm:18.058277777778,athens:23.7163375,oslo:10.722916666667},X={ft:{to_meter:.3048},"us-ft":{to_meter:1200/3937}},H=/[\s_\-\/\(\)]/g,J=function(t){var s,i,a,h={},e=t.split("+").map(function(t){return t.trim()}).filter(function(t){return t}).reduce(function(t,s){var i=s.split("=");return i.push(!0),t[i[0].toLowerCase()]=i[1],t},{}),n={proj:"projName",datum:"datumCode",rf:function(t){h.rf=parseFloat(t)},lat_0:function(t){h.lat0=t*N},lat_1:function(t){h.lat1=t*N},lat_2:function(t){h.lat2=t*N},lat_ts:function(t){h.lat_ts=t*N},lon_0:function(t){h.long0=t*N},lon_1:function(t){h.long1=t*N},lon_2:function(t){h.long2=t*N},alpha:function(t){h.alpha=parseFloat(t)*N},lonc:function(t){h.longc=t*N},x_0:function(t){h.x0=parseFloat(t)},y_0:function(t){h.y0=parseFloat(t)},k_0:function(t){h.k0=parseFloat(t)},k:function(t){h.k0=parseFloat(t)},a:function(t){h.a=parseFloat(t)},b:function(t){h.b=parseFloat(t)},r_a:function(){h.R_A=!0},zone:function(t){h.zone=parseInt(t,10)},south:function(){h.utmSouth=!0},towgs84:function(t){h.datum_params=t.split(",").map(function(t){return parseFloat(t)})},to_meter:function(t){h.to_meter=parseFloat(t)},units:function(t){h.units=t;var s=k(X,t);s&&(h.to_meter=s.to_meter)},from_greenwich:function(t){h.from_greenwich=t*N},pm:function(t){var s=k(W,t);h.from_greenwich=(s||parseFloat(t))*N},nadgrids:function(t){"@null"===t?h.datumCode="none":h.nadgrids=t},axis:function(t){3===t.length&&-1!=="ewnsud".indexOf(t.substr(0,1))&&-1!=="ewnsud".indexOf(t.substr(1,1))&&-1!=="ewnsud".indexOf(t.substr(2,1))&&(h.axis=t)}};for(s in e)i=e[s],s in n?"function"==typeof(a=n[s])?a(i):h[a]=i:h[s]=i;return"string"==typeof h.datumCode&&"WGS84"!==h.datumCode&&(h.datumCode=h.datumCode.toLowerCase()),h},K=1,V=/\s/,Z=/[A-Za-z]/,Y=/[A-Za-z84]/,$=/[,\]]/,tt=/[\d\.E\-\+]/;e.prototype.readCharicter=function(){var t=this.text[this.place++];if(4!==this.state)for(;V.test(t);){if(this.place>=this.text.length)return;t=this.text[this.place++]}switch(this.state){case K:return this.neutral(t);case 2:return this.keyword(t);case 4:return this.quoted(t);case 5:return this.afterquote(t);case 3:return this.number(t);case-1:return}},e.prototype.afterquote=function(t){if('"'===t)return this.word+='"',void(this.state=4);if($.test(t))return this.word=this.word.trim(),void this.afterItem(t);throw new Error("havn't handled \""+t+'" in afterquote yet, index '+this.place)},e.prototype.afterItem=function(t){return","===t?(null!==this.word&&this.currentObject.push(this.word),this.word=null,void(this.state=K)):"]"===t?(this.level--,null!==this.word&&(this.currentObject.push(this.word),this.word=null),this.state=K,this.currentObject=this.stack.pop(),void(this.currentObject||(this.state=-1))):void 0},e.prototype.number=function(t){if(!tt.test(t)){if($.test(t))return this.word=parseFloat(this.word),void this.afterItem(t);throw new Error("havn't handled \""+t+'" in number yet, index '+this.place)}this.word+=t},e.prototype.quoted=function(t){'"'!==t?this.word+=t:this.state=5},e.prototype.keyword=function(t){if(Y.test(t))this.word+=t;else{if("["===t){var s=[];return s.push(this.word),this.level++,null===this.root?this.root=s:this.currentObject.push(s),this.stack.push(this.currentObject),this.currentObject=s,void(this.state=K)}if(!$.test(t))throw new Error("havn't handled \""+t+'" in keyword yet, index '+this.place);this.afterItem(t)}},e.prototype.neutral=function(t){if(Z.test(t))return this.word=t,void(this.state=2);if('"'===t)return this.word="",void(this.state=4);if(tt.test(t))return this.word=t,void(this.state=3);if(!$.test(t))throw new Error("havn't handled \""+t+'" in neutral yet, index '+this.place);this.afterItem(t)},e.prototype.output=function(){for(;this.place<this.text.length;)this.readCharicter();if(-1===this.state)return this.root;throw new Error('unable to parse string "'+this.text+'". State is '+this.state)};var st,it=.017453292519943295,at=function(t){var s=new e(t).output(),i=s.shift(),a=s.shift();s.unshift(["name",a]),s.unshift(["type",i]);var h={};return n(s,h),o(h),h};(st=l)("EPSG:4326","+title=WGS 84 (long/lat) +proj=longlat +ellps=WGS84 +datum=WGS84 +units=degrees"),st("EPSG:4269","+title=NAD83 (long/lat) +proj=longlat +a=6378137.0 +b=6356752.31414036 +ellps=GRS80 +datum=NAD83 +units=degrees"),st("EPSG:3857","+title=WGS 84 / Pseudo-Mercator +proj=merc +a=6378137 +b=6378137 +lat_ts=0.0 +lon_0=0.0 +x_0=0.0 +y_0=0 +k=1.0 +units=m +nadgrids=@null +no_defs"),st.WGS84=st["EPSG:4326"],st["EPSG:3785"]=st["EPSG:3857"],st.GOOGLE=st["EPSG:3857"],st["EPSG:900913"]=st["EPSG:3857"],st["EPSG:102113"]=st["EPSG:3857"];function ht(t,s,i){var a=t*s;return i/Math.sqrt(1-a*a)}function et(t){return t<0?-1:1}function nt(t){return Math.abs(t)<=Q?t:t-et(t)*F}function rt(t,s,i){var a=t*i,h=.5*t,a=Math.pow((1-a)/(1+a),h);return Math.tan(.5*(z-s))/a}function ot(t,s){for(var i,a,h=.5*t,e=z-2*Math.atan(s),n=0;n<=15;n++)if(i=t*Math.sin(e),e+=a=z-2*Math.atan(s*Math.pow((1-i)/(1+i),h))-e,Math.abs(a)<=1e-10)return e;return-9999}var lt=["PROJECTEDCRS","PROJCRS","GEOGCS","GEOCCS","PROJCS","LOCAL_CS","GEODCRS","GEODETICCRS","GEODETICDATUM","ENGCRS","ENGINEERINGCRS"],Mt=["3857","900913","3785","102113"],ct=function(t,s){var i,a;if(t=t||{},!s)return t;for(a in s)void 0!==(i=s[a])&&(t[a]=i);return t},ut=[{init:function(){var t=this.b/this.a;this.es=1-t*t,"x0"in this||(this.x0=0),"y0"in this||(this.y0=0),this.e=Math.sqrt(this.es),this.lat_ts?this.sphere?this.k0=Math.cos(this.lat_ts):this.k0=ht(this.e,Math.sin(this.lat_ts),Math.cos(this.lat_ts)):this.k0||(this.k?this.k0=this.k:this.k0=1)},forward:function(t){var s,i,a,h,e=t.x,n=t.y;return 90<n*B&&n*B<-90&&180<e*B&&e*B<-180||Math.abs(Math.abs(n)-z)<=D?null:(h=this.sphere?(a=this.x0+this.a*this.k0*nt(e-this.long0),this.y0+this.a*this.k0*Math.log(Math.tan(U+.5*n))):(s=Math.sin(n),i=rt(this.e,n,s),a=this.x0+this.a*this.k0*nt(e-this.long0),this.y0-this.a*this.k0*Math.log(i)),t.x=a,t.y=h,t)},inverse:function(t){var s,i,a=t.x-this.x0,h=t.y-this.y0;if(this.sphere)i=z-2*Math.atan(Math.exp(-h/(this.a*this.k0)));else{var e=Math.exp(-h/(this.a*this.k0));if(-9999===(i=ot(this.e,e)))return null}return s=nt(this.long0+a/(this.a*this.k0)),t.x=s,t.y=i,t},names:["Mercator","Popular Visualisation Pseudo Mercator","Mercator_1SP","Mercator_Auxiliary_Sphere","merc"]},{init:function(){},forward:t,inverse:t,names:["longlat","identity"]}],ft={},mt=[],pt={start:function(){ut.forEach(s)},add:s,get:function(t){if(!t)return!1;var s=t.toLowerCase();return void 0!==ft[s]&&mt[ft[s]]?mt[ft[s]]:void 0}},dt={MERIT:{a:6378137,rf:298.257,ellipseName:"MERIT 1983"},SGS85:{a:6378136,rf:298.257,ellipseName:"Soviet Geodetic System 85"},GRS80:{a:6378137,rf:298.257222101,ellipseName:"GRS 1980(IUGG, 1980)"},IAU76:{a:6378140,rf:298.257,ellipseName:"IAU 1976"},airy:{a:6377563.396,b:6356256.91,ellipseName:"Airy 1830"},APL4:{a:6378137,rf:298.25,ellipseName:"Appl. Physics. 1965"},NWL9D:{a:6378145,rf:298.25,ellipseName:"Naval Weapons Lab., 1965"},mod_airy:{a:6377340.189,b:6356034.446,ellipseName:"Modified Airy"},andrae:{a:6377104.43,rf:300,ellipseName:"Andrae 1876 (Den., Iclnd.)"},aust_SA:{a:6378160,rf:298.25,ellipseName:"Australian Natl & S. Amer. 1969"},GRS67:{a:6378160,rf:298.247167427,ellipseName:"GRS 67(IUGG 1967)"},bessel:{a:6377397.155,rf:299.1528128,ellipseName:"Bessel 1841"},bess_nam:{a:6377483.865,rf:299.1528128,ellipseName:"Bessel 1841 (Namibia)"},clrk66:{a:6378206.4,b:6356583.8,ellipseName:"Clarke 1866"},clrk80:{a:6378249.145,rf:293.4663,ellipseName:"Clarke 1880 mod."},clrk58:{a:6378293.645208759,rf:294.2606763692654,ellipseName:"Clarke 1858"},CPM:{a:6375738.7,rf:334.29,ellipseName:"Comm. des Poids et Mesures 1799"},delmbr:{a:6376428,rf:311.5,ellipseName:"Delambre 1810 (Belgium)"},engelis:{a:6378136.05,rf:298.2566,ellipseName:"Engelis 1985"},evrst30:{a:6377276.345,rf:300.8017,ellipseName:"Everest 1830"},evrst48:{a:6377304.063,rf:300.8017,ellipseName:"Everest 1948"},evrst56:{a:6377301.243,rf:300.8017,ellipseName:"Everest 1956"},evrst69:{a:6377295.664,rf:300.8017,ellipseName:"Everest 1969"},evrstSS:{a:6377298.556,rf:300.8017,ellipseName:"Everest (Sabah & Sarawak)"},fschr60:{a:6378166,rf:298.3,ellipseName:"Fischer (Mercury Datum) 1960"},fschr60m:{a:6378155,rf:298.3,ellipseName:"Fischer 1960"},fschr68:{a:6378150,rf:298.3,ellipseName:"Fischer 1968"},helmert:{a:6378200,rf:298.3,ellipseName:"Helmert 1906"},hough:{a:6378270,rf:297,ellipseName:"Hough"},intl:{a:6378388,rf:297,ellipseName:"International 1909 (Hayford)"},kaula:{a:6378163,rf:298.24,ellipseName:"Kaula 1961"},lerch:{a:6378139,rf:298.257,ellipseName:"Lerch 1979"},mprts:{a:6397300,rf:191,ellipseName:"Maupertius 1738"},new_intl:{a:6378157.5,b:6356772.2,ellipseName:"New International 1967"},plessis:{a:6376523,rf:6355863,ellipseName:"Plessis 1817 (France)"},krass:{a:6378245,rf:298.3,ellipseName:"Krassovsky, 1942"},SEasia:{a:6378155,b:6356773.3205,ellipseName:"Southeast Asia"},walbeck:{a:6376896,b:6355834.8467,ellipseName:"Walbeck"},WGS60:{a:6378165,rf:298.3,ellipseName:"WGS 60"},WGS66:{a:6378145,rf:298.25,ellipseName:"WGS 66"},WGS7:{a:6378135,rf:298.26,ellipseName:"WGS 72"}},yt=dt.WGS84={a:6378137,rf:298.257223563,ellipseName:"WGS 84"};dt.sphere={a:6370997,b:6370997,ellipseName:"Normal Sphere (r=6370997)"};var _t={wgs84:{towgs84:"0,0,0",ellipse:"WGS84",datumName:"WGS84"},ch1903:{towgs84:"674.374,15.056,405.346",ellipse:"bessel",datumName:"swiss"},ggrs87:{towgs84:"-199.87,74.79,246.62",ellipse:"GRS80",datumName:"Greek_Geodetic_Reference_System_1987"},nad83:{towgs84:"0,0,0",ellipse:"GRS80",datumName:"North_American_Datum_1983"},nad27:{nadgrids:"@conus,@alaska,@ntv2_0.gsb,@ntv1_can.dat",ellipse:"clrk66",datumName:"North_American_Datum_1927"},potsdam:{towgs84:"606.0,23.0,413.0",ellipse:"bessel",datumName:"Potsdam Rauenberg 1950 DHDN"},carthage:{towgs84:"-263.0,6.0,431.0",ellipse:"clark80",datumName:"Carthage 1934 Tunisia"},hermannskogel:{towgs84:"653.0,-212.0,449.0",ellipse:"bessel",datumName:"Hermannskogel"},osni52:{towgs84:"482.530,-130.596,564.557,-1.042,-0.214,-0.631,8.15",ellipse:"airy",datumName:"Irish National"},ire65:{towgs84:"482.530,-130.596,564.557,-1.042,-0.214,-0.631,8.15",ellipse:"mod_airy",datumName:"Ireland 1965"},rassadiran:{towgs84:"-133.63,-157.5,-158.62",ellipse:"intl",datumName:"Rassadiran"},nzgd49:{towgs84:"59.47,-5.04,187.44,0.47,-0.1,1.024,-4.5993",ellipse:"intl",datumName:"New Zealand Geodetic Datum 1949"},osgb36:{towgs84:"446.448,-125.157,542.060,0.1502,0.2470,0.8421,-20.4894",ellipse:"airy",datumName:"Airy 1830"},s_jtsk:{towgs84:"589,76,480",ellipse:"bessel",datumName:"S-JTSK (Ferro)"},beduaram:{towgs84:"-106,-87,188",ellipse:"clrk80",datumName:"Beduaram"},gunung_segara:{towgs84:"-403,684,41",ellipse:"bessel",datumName:"Gunung Segara Jakarta"},rnb72:{towgs84:"106.869,-52.2978,103.724,-0.33657,0.456955,-1.84218,1",ellipse:"intl",datumName:"Reseau National Belge 1972"}};q.projections=pt,q.projections.start();var xt=function(t,s,i){return h=s,((a=t).datum_type!==h.datum_type||a.a!==h.a||5e-11<Math.abs(a.es-h.es)||(a.datum_type===I?a.datum_params[0]!==h.datum_params[0]||a.datum_params[1]!==h.datum_params[1]||a.datum_params[2]!==h.datum_params[2]:a.datum_type===O&&(a.datum_params[0]!==h.datum_params[0]||a.datum_params[1]!==h.datum_params[1]||a.datum_params[2]!==h.datum_params[2]||a.datum_params[3]!==h.datum_params[3]||a.datum_params[4]!==h.datum_params[4]||a.datum_params[5]!==h.datum_params[5]||a.datum_params[6]!==h.datum_params[6])))&&t.datum_type!==G&&s.datum_type!==G&&(t.es!==s.es||t.a!==s.a||u(t.datum_type)||u(s.datum_type))?(i=M(i,t.es,t.a),u(t.datum_type)&&(i=function(t,s,i){if(s===I)return{x:t.x+i[0],y:t.y+i[1],z:t.z+i[2]};if(s===O){var a=i[0],h=i[1],e=i[2],n=i[3],r=i[4],o=i[5],l=i[6];return{x:l*(t.x-o*t.y+r*t.z)+a,y:l*(o*t.x+t.y-n*t.z)+h,z:l*(-r*t.x+n*t.y+t.z)+e}}}(i,t.datum_type,t.datum_params)),u(s.datum_type)&&(i=function(t,s,i){if(s===I)return{x:t.x-i[0],y:t.y-i[1],z:t.z-i[2]};if(s===O){var a=i[0],h=i[1],e=i[2],n=i[3],r=i[4],o=i[5],l=i[6],M=(t.x-a)/l,c=(t.y-h)/l,u=(t.z-e)/l;return{x:M+o*c-r*u,y:-o*M+c+n*u,z:r*M-n*c+u}}}(i,s.datum_type,s.datum_params)),c(i,s.es,s.a,s.b)):i;var a,h},gt=function(t,s,i){for(var a,h,e=i.x,n=i.y,r=i.z||0,o={},l=0;l<3;l++)if(!s||2!==l||void 0!==i.z)switch(h=0===l?(a=e,-1!=="ew".indexOf(t.axis[l])?"x":"y"):1===l?(a=n,-1!=="ns".indexOf(t.axis[l])?"y":"x"):(a=r,"z"),t.axis[l]){case"e":case"w":case"n":case"s":o[h]=a;break;case"u":void 0!==i[h]&&(o.z=a);break;case"d":void 0!==i[h]&&(o.z=-a);break;default:return null}return o},bt=function(t){var s={x:t[0],y:t[1]};return 2<t.length&&(s.z=t[2]),3<t.length&&(s.m=t[3]),s},vt=function(t){i(t.x),i(t.y)},wt=q("WGS84"),Ct=6,Pt="AJSAJS",St="AFAFAF",Nt=65,kt=73,Et=79,qt=86,It=90,Ot={forward:d,inverse:function(t){var s=g(v(t.toUpperCase()));return s.lat&&s.lon?[s.lon,s.lat,s.lon,s.lat]:[s.left,s.bottom,s.right,s.top]},toPoint:y};C.fromMGRS=function(t){return new C(y(t))},C.prototype.toMGRS=function(t){return d([this.x,this.y],t)};function At(t){var s=[];s[0]=1-t*(.25+t*(.046875+t*(.01953125+t*ts))),s[1]=t*(.75-t*(.046875+t*(.01953125+t*ts)));var i=t*t;return s[2]=i*(.46875-t*(.013020833333333334+.007120768229166667*t)),i*=t,s[3]=i*(.3645833333333333-.005696614583333333*t),s[4]=i*t*.3076171875,s}function Gt(t,s,i,a){return i*=s,s*=s,a[0]*t-i*(a[1]+s*(a[2]+s*(a[3]+s*a[4])))}function jt(t,s,i){for(var a=1/(1-s),h=t,e=20;e;--e){var n=Math.sin(h),r=1-s*n*n;if(h-=r=(Gt(h,n,Math.cos(h),i)-t)*(r*Math.sqrt(r))*a,Math.abs(r)<D)return h}return h}function zt(t){var s=Math.exp(t);return(s-1/s)/2}function Rt(t,s){t=Math.abs(t),s=Math.abs(s);var i=Math.max(t,s),a=Math.min(t,s)/(i||1);return i*Math.sqrt(1+Math.pow(a,2))}function Lt(t){var s,i,a,h=Math.abs(t);return s=h*(1+h/(Rt(1,h)+1)),h=0==(a=(i=1+s)-1)?s:s*Math.log(i)/a,t<0?-h:h}function Tt(t,s){for(var i,a=2*Math.cos(2*s),h=t.length-1,e=t[h],n=0;0<=--h;)i=a*e-n+t[h],n=e,e=i;return s+i*Math.sin(2*s)}function Dt(t,s,i){for(var a,h,e,n,r=Math.sin(s),o=Math.cos(s),l=zt(i),M=(e=i,((n=Math.exp(e))+1/n)/2),c=2*o*M,u=-2*r*l,f=t.length-1,m=t[f],p=0,d=0,y=0;0<=--f;)a=d,h=p,m=c*(d=m)-a-u*(p=y)+t[f],y=u*d-h+c*p;return[(c=r*M)*m-(u=o*l)*y,c*y+u*m]}function Bt(t,s){return Math.pow((1-t)/(1+t),s)}function Ut(t,s,i,a,h){return t*h-s*Math.sin(2*h)+i*Math.sin(4*h)-a*Math.sin(6*h)}function Ft(t){return 1-.25*t*(1+t/16*(3+1.25*t))}function Qt(t){return.375*t*(1+.25*t*(1+.46875*t))}function Wt(t){return.05859375*t*t*(1+.75*t)}function Xt(t){return t*t*t*(35/3072)}function Ht(t,s,i){var a=s*i;return t/Math.sqrt(1-a*a)}function Jt(t){return Math.abs(t)<z?t:t-et(t)*Math.PI}function Kt(t,s,i,a,h){for(var e,n=t/s,r=0;r<15;r++)if(n+=e=(t-(s*n-i*Math.sin(2*n)+a*Math.sin(4*n)-h*Math.sin(6*n)))/(s-2*i*Math.cos(2*n)+4*a*Math.cos(4*n)-6*h*Math.cos(6*n)),Math.abs(e)<=1e-10)return n;return NaN}function Vt(t,s){var i;return 1e-7<t?(1-t*t)*(s/(1-(i=t*s)*i)-.5/t*Math.log((1-i)/(1+i))):2*s}function Zt(t){return 1<Math.abs(t)&&(t=1<t?1:-1),Math.asin(t)}function Yt(t,s){return t[0]+s*(t[1]+s*(t[2]+s*t[3]))}var $t,ts=.01068115234375,ss={init:function(){this.x0=void 0!==this.x0?this.x0:0,this.y0=void 0!==this.y0?this.y0:0,this.long0=void 0!==this.long0?this.long0:0,this.lat0=void 0!==this.lat0?this.lat0:0,this.es&&(this.en=At(this.es),this.ml0=Gt(this.lat0,Math.sin(this.lat0),Math.cos(this.lat0),this.en))},forward:function(t){var s=t.x,i=t.y,a=nt(s-this.long0),h=Math.sin(i),e=Math.cos(i);if(this.es){var n=e*a,r=Math.pow(n,2),o=this.ep2*Math.pow(e,2),l=Math.pow(o,2),M=Math.abs(e)>D?Math.tan(i):0,c=Math.pow(M,2),u=Math.pow(c,2),f=1-this.es*Math.pow(h,2);n/=Math.sqrt(f);var m=Gt(i,h,e,this.en),p=this.a*(this.k0*n*(1+r/6*(1-c+o+r/20*(5-18*c+u+14*o-58*c*o+r/42*(61+179*u-u*c-479*c)))))+this.x0,d=this.a*(this.k0*(m-this.ml0+h*a*n/2*(1+r/12*(5-c+9*o+4*l+r/30*(61+u-58*c+270*o-330*c*o+r/56*(1385+543*u-u*c-3111*c))))))+this.y0}else{var y=e*Math.sin(a);if(Math.abs(Math.abs(y)-1)<D)return 93;if(p=.5*this.a*this.k0*Math.log((1+y)/(1-y))+this.x0,d=e*Math.cos(a)/Math.sqrt(1-Math.pow(y,2)),1<=(y=Math.abs(d))){if(D<y-1)return 93;d=0}else d=Math.acos(d);i<0&&(d=-d),d=this.a*this.k0*(d-this.lat0)+this.y0}return t.x=p,t.y=d,t},inverse:function(t){var s,i,a,h,e,n,r,o,l,M,c,u,f,m,p,d,y,_=(t.x-this.x0)*(1/this.a),x=(t.y-this.y0)*(1/this.a);return f=this.es?(l=this.ml0+x/this.k0,s=jt(l,this.es,this.en),Math.abs(s)<z?(i=Math.sin(s),a=Math.cos(s),h=Math.abs(a)>D?Math.tan(s):0,e=this.ep2*Math.pow(a,2),n=Math.pow(e,2),r=Math.pow(h,2),o=Math.pow(r,2),l=1-this.es*Math.pow(i,2),M=_*Math.sqrt(l)/this.k0,u=s-(l*=h)*(c=Math.pow(M,2))/(1-this.es)*.5*(1-c/12*(5+3*r-9*e*r+e-4*n-c/30*(61+90*r-252*e*r+45*o+46*e-c/56*(1385+3633*r+4095*o+1574*o*r)))),nt(this.long0+M*(1-c/6*(1+2*r+e-c/20*(5+28*r+24*o+8*e*r+6*e-c/42*(61+662*r+1320*o+720*o*r))))/a)):(u=z*et(x),0)):(p=.5*((m=Math.exp(_/this.k0))-1/m),d=this.lat0+x/this.k0,y=Math.cos(d),l=Math.sqrt((1-Math.pow(y,2))/(1+Math.pow(p,2))),u=Math.asin(l),x<0&&(u=-u),0==p&&0===y?0:nt(Math.atan2(p,y)+this.long0)),t.x=f,t.y=u,t},names:["Transverse_Mercator","Transverse Mercator","tmerc"]},is={init:function(){if(void 0===this.es||this.es<=0)throw new Error("incorrect elliptical usage");this.x0=void 0!==this.x0?this.x0:0,this.y0=void 0!==this.y0?this.y0:0,this.long0=void 0!==this.long0?this.long0:0,this.lat0=void 0!==this.lat0?this.lat0:0,this.cgb=[],this.cbg=[],this.utg=[],this.gtu=[];var t=this.es/(1+Math.sqrt(1-this.es)),s=t/(2-t),i=s;this.cgb[0]=s*(2+s*(-2/3+s*(s*(116/45+s*(26/45+-2854/675*s))-2))),this.cbg[0]=s*(s*(2/3+s*(4/3+s*(-82/45+s*(32/45+4642/4725*s))))-2),i*=s,this.cgb[1]=i*(7/3+s*(s*(-227/45+s*(2704/315+2323/945*s))-1.6)),this.cbg[1]=i*(5/3+s*(-16/15+s*(-13/9+s*(904/315+-1522/945*s)))),i*=s,this.cgb[2]=i*(56/15+s*(-136/35+s*(-1262/105+73814/2835*s))),this.cbg[2]=i*(-26/15+s*(34/21+s*(1.6+-12686/2835*s))),i*=s,this.cgb[3]=i*(4279/630+s*(-332/35+-399572/14175*s)),this.cbg[3]=i*(1237/630+s*(-24832/14175*s-2.4)),i*=s,this.cgb[4]=i*(4174/315+-144838/6237*s),this.cbg[4]=i*(-734/315+109598/31185*s),i*=s,this.cgb[5]=i*(601676/22275),this.cbg[5]=i*(444337/155925),i=Math.pow(s,2),this.Qn=this.k0/(1+s)*(1+i*(.25+i*(1/64+i/256))),this.utg[0]=s*(s*(2/3+s*(-37/96+s*(1/360+s*(81/512+-96199/604800*s))))-.5),this.gtu[0]=s*(.5+s*(-2/3+s*(5/16+s*(41/180+s*(-127/288+7891/37800*s))))),this.utg[1]=i*(-1/48+s*(-1/15+s*(437/1440+s*(-46/105+1118711/3870720*s)))),this.gtu[1]=i*(13/48+s*(s*(557/1440+s*(281/630+-1983433/1935360*s))-.6)),i*=s,this.utg[2]=i*(-17/480+s*(37/840+s*(209/4480+-5569/90720*s))),this.gtu[2]=i*(61/240+s*(-103/140+s*(15061/26880+167603/181440*s))),i*=s,this.utg[3]=i*(-4397/161280+s*(11/504+830251/7257600*s)),this.gtu[3]=i*(49561/161280+s*(-179/168+6601661/7257600*s)),i*=s,this.utg[4]=i*(-4583/161280+108847/3991680*s),this.gtu[4]=i*(34729/80640+-3418889/1995840*s),i*=s,this.utg[5]=-.03233083094085698*i,this.gtu[5]=.6650675310896665*i;var a=Tt(this.cbg,this.lat0);this.Zb=-this.Qn*(a+function(t,s){for(var i,a=2*Math.cos(s),h=t.length-1,e=t[h],n=0;0<=--h;)i=a*e-n+t[h],n=e,e=i;return Math.sin(s)*i}(this.gtu,2*a))},forward:function(t){var s=nt(t.x-this.long0),i=t.y,i=Tt(this.cbg,i),a=Math.sin(i),h=Math.cos(i),e=Math.sin(s),n=Math.cos(s);i=Math.atan2(a,n*h),s=Math.atan2(e*h,Rt(a,h*n)),s=Lt(Math.tan(s));var r,o,l=Dt(this.gtu,2*i,2*s);return i+=l[0],s+=l[1],o=Math.abs(s)<=2.623395162778?(r=this.a*(this.Qn*s)+this.x0,this.a*(this.Qn*i+this.Zb)+this.y0):r=1/0,t.x=r,t.y=o,t},inverse:function(t){var s,i,a,h,e,n,r,o=(t.x-this.x0)*(1/this.a),l=(t.y-this.y0)*(1/this.a);return l=(l-this.Zb)/this.Qn,o/=this.Qn,r=Math.abs(o)<=2.623395162778?(l+=(s=Dt(this.utg,2*l,2*o))[0],o+=s[1],o=Math.atan(zt(o)),i=Math.sin(l),a=Math.cos(l),h=Math.sin(o),e=Math.cos(o),l=Math.atan2(i*e,Rt(h,e*a)),o=Math.atan2(h,e*a),n=nt(o+this.long0),Tt(this.cgb,l)):n=1/0,t.x=n,t.y=r,t},names:["Extended_Transverse_Mercator","Extended Transverse Mercator","etmerc"]},as={init:function(){var t=function(t,s){if(void 0===t){if((t=Math.floor(30*(nt(s)+Math.PI)/Math.PI)+1)<0)return 0;if(60<t)return 60}return t}(this.zone,this.long0);if(void 0===t)throw new Error("unknown utm zone");this.lat0=0,this.long0=(6*Math.abs(t)-183)*N,this.x0=5e5,this.y0=this.utmSouth?1e7:0,this.k0=.9996,is.init.apply(this),this.forward=is.forward,this.inverse=is.inverse},names:["Universal Transverse Mercator System","utm"],dependsOn:"etmerc"},hs={init:function(){var t=Math.sin(this.lat0),s=Math.cos(this.lat0);s*=s,this.rc=Math.sqrt(1-this.es)/(1-this.es*t*t),this.C=Math.sqrt(1+this.es*s*s/(1-this.es)),this.phic0=Math.asin(t/this.C),this.ratexp=.5*this.C*this.e,this.K=Math.tan(.5*this.phic0+U)/(Math.pow(Math.tan(.5*this.lat0+U),this.C)*Bt(this.e*t,this.ratexp))},forward:function(t){var s=t.x,i=t.y;return t.y=2*Math.atan(this.K*Math.pow(Math.tan(.5*i+U),this.C)*Bt(this.e*Math.sin(i),this.ratexp))-z,t.x=this.C*s,t},inverse:function(t){for(var s=t.x/this.C,i=t.y,a=Math.pow(Math.tan(.5*i+U)/this.K,1/this.C),h=20;0<h&&(i=2*Math.atan(a*Bt(this.e*Math.sin(t.y),-.5*this.e))-z,!(Math.abs(i-t.y)<1e-14));--h)t.y=i;return h?(t.x=s,t.y=i,t):null},names:["gauss"]},es={init:function(){hs.init.apply(this),this.rc&&(this.sinc0=Math.sin(this.phic0),this.cosc0=Math.cos(this.phic0),this.R2=2*this.rc,this.title||(this.title="Oblique Stereographic Alternative"))},forward:function(t){var s,i,a,h;return t.x=nt(t.x-this.long0),hs.forward.apply(this,[t]),s=Math.sin(t.y),i=Math.cos(t.y),a=Math.cos(t.x),h=this.k0*this.R2/(1+this.sinc0*s+this.cosc0*i*a),t.x=h*i*Math.sin(t.x),t.y=h*(this.cosc0*s-this.sinc0*i*a),t.x=this.a*t.x+this.x0,t.y=this.a*t.y+this.y0,t},inverse:function(t){var s,i,a,h,e,n;return t.x=(t.x-this.x0)/this.a,t.y=(t.y-this.y0)/this.a,t.x/=this.k0,t.y/=this.k0,n=(s=Math.sqrt(t.x*t.x+t.y*t.y))?(i=2*Math.atan2(s,this.R2),a=Math.sin(i),h=Math.cos(i),e=Math.asin(h*this.sinc0+t.y*a*this.cosc0/s),Math.atan2(t.x*a,s*this.cosc0*h-t.y*this.sinc0*a)):(e=this.phic0,0),t.x=n,t.y=e,hs.inverse.apply(this,[t]),t.x=nt(t.x+this.long0),t},names:["Stereographic_North_Pole","Oblique_Stereographic","Polar_Stereographic","sterea","Oblique Stereographic Alternative","Double_Stereographic"]},ns={init:function(){this.coslat0=Math.cos(this.lat0),this.sinlat0=Math.sin(this.lat0),this.sphere?1===this.k0&&!isNaN(this.lat_ts)&&Math.abs(this.coslat0)<=D&&(this.k0=.5*(1+et(this.lat0)*Math.sin(this.lat_ts))):(Math.abs(this.coslat0)<=D&&(0<this.lat0?this.con=1:this.con=-1),this.cons=Math.sqrt(Math.pow(1+this.e,1+this.e)*Math.pow(1-this.e,1-this.e)),1===this.k0&&!isNaN(this.lat_ts)&&Math.abs(this.coslat0)<=D&&(this.k0=.5*this.cons*ht(this.e,Math.sin(this.lat_ts),Math.cos(this.lat_ts))/rt(this.e,this.con*this.lat_ts,this.con*Math.sin(this.lat_ts))),this.ms1=ht(this.e,this.sinlat0,this.coslat0),this.X0=2*Math.atan(this.ssfn_(this.lat0,this.sinlat0,this.e))-z,this.cosX0=Math.cos(this.X0),this.sinX0=Math.sin(this.X0))},forward:function(t){var s,i,a,h,e,n,r=t.x,o=t.y,l=Math.sin(o),M=Math.cos(o),c=nt(r-this.long0);return Math.abs(Math.abs(r-this.long0)-Math.PI)<=D&&Math.abs(o+this.lat0)<=D?(t.x=NaN,t.y=NaN):this.sphere?(s=2*this.k0/(1+this.sinlat0*l+this.coslat0*M*Math.cos(c)),t.x=this.a*s*M*Math.sin(c)+this.x0,t.y=this.a*s*(this.coslat0*l-this.sinlat0*M*Math.cos(c))+this.y0):(i=2*Math.atan(this.ssfn_(o,l,this.e))-z,h=Math.cos(i),a=Math.sin(i),Math.abs(this.coslat0)<=D?(e=rt(this.e,o*this.con,this.con*l),n=2*this.a*this.k0*e/this.cons,t.x=this.x0+n*Math.sin(r-this.long0),t.y=this.y0-this.con*n*Math.cos(r-this.long0)):(Math.abs(this.sinlat0)<D?(s=2*this.a*this.k0/(1+h*Math.cos(c)),t.y=s*a):(s=2*this.a*this.k0*this.ms1/(this.cosX0*(1+this.sinX0*a+this.cosX0*h*Math.cos(c))),t.y=s*(this.cosX0*a-this.sinX0*h*Math.cos(c))+this.y0),t.x=s*h*Math.sin(c)+this.x0)),t},inverse:function(t){t.x-=this.x0,t.y-=this.y0;var s,i,a,h=Math.sqrt(t.x*t.x+t.y*t.y);if(this.sphere){var e=2*Math.atan(h/(2*this.a*this.k0)),n=this.long0,r=this.lat0;return h<=D||(r=Math.asin(Math.cos(e)*this.sinlat0+t.y*Math.sin(e)*this.coslat0/h),n=nt(Math.abs(this.coslat0)<D?0<this.lat0?this.long0+Math.atan2(t.x,-1*t.y):this.long0+Math.atan2(t.x,t.y):this.long0+Math.atan2(t.x*Math.sin(e),h*this.coslat0*Math.cos(e)-t.y*this.sinlat0*Math.sin(e)))),t.x=n,t.y=r,t}if(Math.abs(this.coslat0)<=D){if(h<=D)return r=this.lat0,n=this.long0,t.x=n,t.y=r,t;t.x*=this.con,t.y*=this.con,s=h*this.cons/(2*this.a*this.k0),r=this.con*ot(this.e,s),n=this.con*nt(this.con*this.long0+Math.atan2(t.x,-1*t.y))}else i=2*Math.atan(h*this.cosX0/(2*this.a*this.k0*this.ms1)),n=this.long0,h<=D?a=this.X0:(a=Math.asin(Math.cos(i)*this.sinX0+t.y*Math.sin(i)*this.cosX0/h),n=nt(this.long0+Math.atan2(t.x*Math.sin(i),h*this.cosX0*Math.cos(i)-t.y*this.sinX0*Math.sin(i)))),r=-1*ot(this.e,Math.tan(.5*(z+a)));return t.x=n,t.y=r,t},names:["stere","Stereographic_South_Pole","Polar Stereographic (variant B)"],ssfn_:function(t,s,i){return s*=i,Math.tan(.5*(z+t))*Math.pow((1-s)/(1+s),.5*i)}},rs={init:function(){var t=this.lat0;this.lambda0=this.long0;var s=Math.sin(t),i=this.a,a=1/this.rf,h=2*a-Math.pow(a,2),e=this.e=Math.sqrt(h);this.R=this.k0*i*Math.sqrt(1-h)/(1-h*Math.pow(s,2)),this.alpha=Math.sqrt(1+h/(1-h)*Math.pow(Math.cos(t),4)),this.b0=Math.asin(s/this.alpha);var n=Math.log(Math.tan(Math.PI/4+this.b0/2)),r=Math.log(Math.tan(Math.PI/4+t/2)),o=Math.log((1+e*s)/(1-e*s));this.K=n-this.alpha*r+this.alpha*e/2*o},forward:function(t){var s=Math.log(Math.tan(Math.PI/4-t.y/2)),i=this.e/2*Math.log((1+this.e*Math.sin(t.y))/(1-this.e*Math.sin(t.y))),a=-this.alpha*(s+i)+this.K,h=2*(Math.atan(Math.exp(a))-Math.PI/4),e=this.alpha*(t.x-this.lambda0),n=Math.atan(Math.sin(e)/(Math.sin(this.b0)*Math.tan(h)+Math.cos(this.b0)*Math.cos(e))),r=Math.asin(Math.cos(this.b0)*Math.sin(h)-Math.sin(this.b0)*Math.cos(h)*Math.cos(e));return t.y=this.R/2*Math.log((1+Math.sin(r))/(1-Math.sin(r)))+this.y0,t.x=this.R*n+this.x0,t},inverse:function(t){for(var s=t.x-this.x0,i=t.y-this.y0,a=s/this.R,h=2*(Math.atan(Math.exp(i/this.R))-Math.PI/4),e=Math.asin(Math.cos(this.b0)*Math.sin(h)+Math.sin(this.b0)*Math.cos(h)*Math.cos(a)),n=Math.atan(Math.sin(a)/(Math.cos(this.b0)*Math.cos(a)-Math.sin(this.b0)*Math.tan(h))),r=this.lambda0+n/this.alpha,o=0,l=e,M=-1e3,c=0;1e-7<Math.abs(l-M);){if(20<++c)return;o=1/this.alpha*(Math.log(Math.tan(Math.PI/4+e/2))-this.K)+this.e*Math.log(Math.tan(Math.PI/4+Math.asin(this.e*Math.sin(l))/2)),M=l,l=2*Math.atan(Math.exp(o))-Math.PI/2}return t.x=r,t.y=l,t},names:["somerc"]},os={init:function(){this.no_off=this.no_off||!1,this.no_rot=this.no_rot||!1,isNaN(this.k0)&&(this.k0=1);var t=Math.sin(this.lat0),s=Math.cos(this.lat0),i=this.e*t;this.bl=Math.sqrt(1+this.es/(1-this.es)*Math.pow(s,4)),this.al=this.a*this.bl*this.k0*Math.sqrt(1-this.es)/(1-i*i);var a,h,e,n,r,o,l,M,c,u,f=rt(this.e,this.lat0,t),m=this.bl/s*Math.sqrt((1-this.es)/(1-i*i));m*m<1&&(m=1),isNaN(this.longc)?(h=rt(this.e,this.lat1,Math.sin(this.lat1)),e=rt(this.e,this.lat2,Math.sin(this.lat2)),0<=this.lat0?this.el=(m+Math.sqrt(m*m-1))*Math.pow(f,this.bl):this.el=(m-Math.sqrt(m*m-1))*Math.pow(f,this.bl),n=Math.pow(h,this.bl),r=Math.pow(e,this.bl),o=.5*((a=this.el/n)-1/a),l=(this.el*this.el-r*n)/(this.el*this.el+r*n),M=(r-n)/(r+n),c=nt(this.long1-this.long2),this.long0=.5*(this.long1+this.long2)-Math.atan(l*Math.tan(.5*this.bl*c)/M)/this.bl,this.long0=nt(this.long0),u=nt(this.long1-this.long0),this.gamma0=Math.atan(Math.sin(this.bl*u)/o),this.alpha=Math.asin(m*Math.sin(this.gamma0))):(a=0<=this.lat0?m+Math.sqrt(m*m-1):m-Math.sqrt(m*m-1),this.el=a*Math.pow(f,this.bl),o=.5*(a-1/a),this.gamma0=Math.asin(Math.sin(this.alpha)/m),this.long0=this.longc-Math.asin(o*Math.tan(this.gamma0))/this.bl),this.no_off?this.uc=0:0<=this.lat0?this.uc=this.al/this.bl*Math.atan2(Math.sqrt(m*m-1),Math.cos(this.alpha)):this.uc=-1*this.al/this.bl*Math.atan2(Math.sqrt(m*m-1),Math.cos(this.alpha))},forward:function(t){var s,i,a,h,e,n,r,o,l,M=t.x,c=t.y,u=nt(M-this.long0);return l=Math.abs(Math.abs(c)-z)<=D?(s=0<c?-1:1,o=this.al/this.bl*Math.log(Math.tan(U+s*this.gamma0*.5)),-1*s*z*this.al/this.bl):(i=rt(this.e,c,Math.sin(c)),h=.5*((a=this.el/Math.pow(i,this.bl))-1/a),e=.5*(a+1/a),n=Math.sin(this.bl*u),r=(h*Math.sin(this.gamma0)-n*Math.cos(this.gamma0))/e,o=Math.abs(Math.abs(r)-1)<=D?Number.POSITIVE_INFINITY:.5*this.al*Math.log((1-r)/(1+r))/this.bl,Math.abs(Math.cos(this.bl*u))<=D?this.al*this.bl*u:this.al*Math.atan2(h*Math.cos(this.gamma0)+n*Math.sin(this.gamma0),Math.cos(this.bl*u))/this.bl),this.no_rot?(t.x=this.x0+l,t.y=this.y0+o):(l-=this.uc,t.x=this.x0+o*Math.cos(this.alpha)+l*Math.sin(this.alpha),t.y=this.y0+l*Math.cos(this.alpha)-o*Math.sin(this.alpha)),t},inverse:function(t){var s,i;this.no_rot?(i=t.y-this.y0,s=t.x-this.x0):(i=(t.x-this.x0)*Math.cos(this.alpha)-(t.y-this.y0)*Math.sin(this.alpha),s=(t.y-this.y0)*Math.cos(this.alpha)+(t.x-this.x0)*Math.sin(this.alpha),s+=this.uc);var a=Math.exp(-1*this.bl*i/this.al),h=.5*(a-1/a),e=.5*(a+1/a),n=Math.sin(this.bl*s/this.al),r=(n*Math.cos(this.gamma0)+h*Math.sin(this.gamma0))/e,o=Math.pow(this.el/Math.sqrt((1+r)/(1-r)),1/this.bl);return Math.abs(r-1)<D?(t.x=this.long0,t.y=z):Math.abs(1+r)<D?(t.x=this.long0,t.y=-1*z):(t.y=ot(this.e,o),t.x=nt(this.long0-Math.atan2(h*Math.cos(this.gamma0)-n*Math.sin(this.gamma0),Math.cos(this.bl*s/this.al))/this.bl)),t},names:["Hotine_Oblique_Mercator","Hotine Oblique Mercator","Hotine_Oblique_Mercator_Azimuth_Natural_Origin","Hotine_Oblique_Mercator_Azimuth_Center","omerc"]},ls={init:function(){var t,s,i,a,h,e,n,r,o,l;this.lat2||(this.lat2=this.lat1),this.k0||(this.k0=1),this.x0=this.x0||0,this.y0=this.y0||0,Math.abs(this.lat1+this.lat2)<D||(t=this.b/this.a,this.e=Math.sqrt(1-t*t),s=Math.sin(this.lat1),i=Math.cos(this.lat1),a=ht(this.e,s,i),h=rt(this.e,this.lat1,s),e=Math.sin(this.lat2),n=Math.cos(this.lat2),r=ht(this.e,e,n),o=rt(this.e,this.lat2,e),l=rt(this.e,this.lat0,Math.sin(this.lat0)),Math.abs(this.lat1-this.lat2)>D?this.ns=Math.log(a/r)/Math.log(h/o):this.ns=s,isNaN(this.ns)&&(this.ns=s),this.f0=a/(this.ns*Math.pow(h,this.ns)),this.rh=this.a*this.f0*Math.pow(l,this.ns),this.title||(this.title="Lambert Conformal Conic"))},forward:function(t){var s=t.x,i=t.y;Math.abs(2*Math.abs(i)-Math.PI)<=D&&(i=et(i)*(z-2*D));var a,h,e=Math.abs(Math.abs(i)-z);if(D<e)a=rt(this.e,i,Math.sin(i)),h=this.a*this.f0*Math.pow(a,this.ns);else{if((e=i*this.ns)<=0)return null;h=0}var n=this.ns*nt(s-this.long0);return t.x=this.k0*(h*Math.sin(n))+this.x0,t.y=this.k0*(this.rh-h*Math.cos(n))+this.y0,t},inverse:function(t){var s,i,a,h,e=(t.x-this.x0)/this.k0,n=this.rh-(t.y-this.y0)/this.k0,r=0<this.ns?(s=Math.sqrt(e*e+n*n),1):(s=-Math.sqrt(e*e+n*n),-1),o=0;if(0!==s&&(o=Math.atan2(r*e,r*n)),0!==s||0<this.ns){if(r=1/this.ns,i=Math.pow(s/(this.a*this.f0),r),-9999===(a=ot(this.e,i)))return null}else a=-z;return h=nt(o/this.ns+this.long0),t.x=h,t.y=a,t},names:["Lambert Tangential Conformal Conic Projection","Lambert_Conformal_Conic","Lambert_Conformal_Conic_2SP","lcc"]},Ms={init:function(){this.a=6377397.155,this.es=.006674372230614,this.e=Math.sqrt(this.es),this.lat0||(this.lat0=.863937979737193),this.long0||(this.long0=.4334234309119251),this.k0||(this.k0=.9999),this.s45=.785398163397448,this.s90=2*this.s45,this.fi0=this.lat0,this.e2=this.es,this.e=Math.sqrt(this.e2),this.alfa=Math.sqrt(1+this.e2*Math.pow(Math.cos(this.fi0),4)/(1-this.e2)),this.uq=1.04216856380474,this.u0=Math.asin(Math.sin(this.fi0)/this.alfa),this.g=Math.pow((1+this.e*Math.sin(this.fi0))/(1-this.e*Math.sin(this.fi0)),this.alfa*this.e/2),this.k=Math.tan(this.u0/2+this.s45)/Math.pow(Math.tan(this.fi0/2+this.s45),this.alfa)*this.g,this.k1=this.k0,this.n0=this.a*Math.sqrt(1-this.e2)/(1-this.e2*Math.pow(Math.sin(this.fi0),2)),this.s0=1.37008346281555,this.n=Math.sin(this.s0),this.ro0=this.k1*this.n0/Math.tan(this.s0),this.ad=this.s90-this.uq},forward:function(t){var s=t.x,i=t.y,a=nt(s-this.long0),h=Math.pow((1+this.e*Math.sin(i))/(1-this.e*Math.sin(i)),this.alfa*this.e/2),e=2*(Math.atan(this.k*Math.pow(Math.tan(i/2+this.s45),this.alfa)/h)-this.s45),n=-a*this.alfa,r=Math.asin(Math.cos(this.ad)*Math.sin(e)+Math.sin(this.ad)*Math.cos(e)*Math.cos(n)),o=Math.asin(Math.cos(e)*Math.sin(n)/Math.cos(r)),l=this.n*o,M=this.ro0*Math.pow(Math.tan(this.s0/2+this.s45),this.n)/Math.pow(Math.tan(r/2+this.s45),this.n);return t.y=M*Math.cos(l),t.x=M*Math.sin(l),this.czech||(t.y*=-1,t.x*=-1),t},inverse:function(t){var s,i,a,h,e,n,r,o=t.x;t.x=t.y,t.y=o,this.czech||(t.y*=-1,t.x*=-1),e=Math.sqrt(t.x*t.x+t.y*t.y),h=Math.atan2(t.y,t.x)/Math.sin(this.s0),a=2*(Math.atan(Math.pow(this.ro0/e,1/this.n)*Math.tan(this.s0/2+this.s45))-this.s45),s=Math.asin(Math.cos(this.ad)*Math.sin(a)-Math.sin(this.ad)*Math.cos(a)*Math.cos(h)),i=Math.asin(Math.cos(a)*Math.sin(h)/Math.cos(s)),t.x=this.long0-i/this.alfa,n=s;for(var l=r=0;t.y=2*(Math.atan(Math.pow(this.k,-1/this.alfa)*Math.pow(Math.tan(s/2+this.s45),1/this.alfa)*Math.pow((1+this.e*Math.sin(n))/(1-this.e*Math.sin(n)),this.e/2))-this.s45),Math.abs(n-t.y)<1e-10&&(r=1),n=t.y,l+=1,0===r&&l<15;);return 15<=l?null:t},names:["Krovak","krovak"]},cs={init:function(){this.sphere||(this.e0=Ft(this.es),this.e1=Qt(this.es),this.e2=Wt(this.es),this.e3=Xt(this.es),this.ml0=this.a*Ut(this.e0,this.e1,this.e2,this.e3,this.lat0))},forward:function(t){var s,i,a,h,e,n,r,o,l,M=t.x,c=t.y,M=nt(M-this.long0);return l=this.sphere?(o=this.a*Math.asin(Math.cos(c)*Math.sin(M)),this.a*(Math.atan2(Math.tan(c),Math.cos(M))-this.lat0)):(s=Math.sin(c),i=Math.cos(c),a=Ht(this.a,this.e,s),h=Math.tan(c)*Math.tan(c),o=a*(e=M*Math.cos(c))*(1-(n=e*e)*h*(1/6-(8-h+8*(r=this.es*i*i/(1-this.es)))*n/120)),this.a*Ut(this.e0,this.e1,this.e2,this.e3,c)-this.ml0+a*s/i*n*(.5+(5-h+6*r)*n/24)),t.x=o+this.x0,t.y=l+this.y0,t},inverse:function(t){t.x-=this.x0,t.y-=this.y0;var s=t.x/this.a,i=t.y/this.a;if(this.sphere)var a=i+this.lat0,h=Math.asin(Math.sin(a)*Math.cos(s)),e=Math.atan2(Math.tan(s),Math.cos(a));else{var n=this.ml0/this.a+i,r=Kt(n,this.e0,this.e1,this.e2,this.e3);if(Math.abs(Math.abs(r)-z)<=D)return t.x=this.long0,t.y=z,i<0&&(t.y*=-1),t;var o=Ht(this.a,this.e,Math.sin(r)),l=o*o*o/this.a/this.a*(1-this.es),M=Math.pow(Math.tan(r),2),c=s*this.a/o,u=c*c;h=r-o*Math.tan(r)/l*c*c*(.5-(1+3*M)*c*c/24),e=c*(1-u*(M/3+(1+3*M)*M*u/15))/Math.cos(r)}return t.x=nt(e+this.long0),t.y=Jt(h),t},names:["Cassini","Cassini_Soldner","cass"]},us={init:function(){var t,s,i,a,h=Math.abs(this.lat0);if(Math.abs(h-z)<D?this.mode=this.lat0<0?this.S_POLE:this.N_POLE:Math.abs(h)<D?this.mode=this.EQUIT:this.mode=this.OBLIQ,0<this.es)switch(this.qp=Vt(this.e,1),this.mmf=.5/(1-this.es),this.apa=(s=this.es,(a=[])[0]=.3333333333333333*s,i=s*s,a[0]+=.17222222222222222*i,a[1]=.06388888888888888*i,i*=s,a[0]+=.10257936507936508*i,a[1]+=.0664021164021164*i,a[2]=.016415012942191543*i,a),this.mode){case this.N_POLE:case this.S_POLE:this.dd=1;break;case this.EQUIT:this.rq=Math.sqrt(.5*this.qp),this.dd=1/this.rq,this.xmf=1,this.ymf=.5*this.qp;break;case this.OBLIQ:this.rq=Math.sqrt(.5*this.qp),t=Math.sin(this.lat0),this.sinb1=Vt(this.e,t)/this.qp,this.cosb1=Math.sqrt(1-this.sinb1*this.sinb1),this.dd=Math.cos(this.lat0)/(Math.sqrt(1-this.es*t*t)*this.rq*this.cosb1),this.ymf=(this.xmf=this.rq)/this.dd,this.xmf*=this.dd}else this.mode===this.OBLIQ&&(this.sinph0=Math.sin(this.lat0),this.cosph0=Math.cos(this.lat0))},forward:function(t){var s,i,a,h,e,n,r,o,l,M,c=t.x,u=t.y,c=nt(c-this.long0);if(this.sphere){if(e=Math.sin(u),M=Math.cos(u),a=Math.cos(c),this.mode===this.OBLIQ||this.mode===this.EQUIT){if((i=this.mode===this.EQUIT?1+M*a:1+this.sinph0*e+this.cosph0*M*a)<=D)return null;s=(i=Math.sqrt(2/i))*M*Math.sin(c),i*=this.mode===this.EQUIT?e:this.cosph0*e-this.sinph0*M*a}else if(this.mode===this.N_POLE||this.mode===this.S_POLE){if(this.mode===this.N_POLE&&(a=-a),Math.abs(u+this.lat0)<D)return null;i=U-.5*u,s=(i=2*(this.mode===this.S_POLE?Math.cos(i):Math.sin(i)))*Math.sin(c),i*=a}}else{switch(l=o=r=0,a=Math.cos(c),h=Math.sin(c),e=Math.sin(u),n=Vt(this.e,e),this.mode!==this.OBLIQ&&this.mode!==this.EQUIT||(r=n/this.qp,o=Math.sqrt(1-r*r)),this.mode){case this.OBLIQ:l=1+this.sinb1*r+this.cosb1*o*a;break;case this.EQUIT:l=1+o*a;break;case this.N_POLE:l=z+u,n=this.qp-n;break;case this.S_POLE:l=u-z,n=this.qp+n}if(Math.abs(l)<D)return null;switch(this.mode){case this.OBLIQ:case this.EQUIT:l=Math.sqrt(2/l),i=this.mode===this.OBLIQ?this.ymf*l*(this.cosb1*r-this.sinb1*o*a):(l=Math.sqrt(2/(1+o*a)))*r*this.ymf,s=this.xmf*l*o*h;break;case this.N_POLE:case this.S_POLE:0<=n?(s=(l=Math.sqrt(n))*h,i=a*(this.mode===this.S_POLE?l:-l)):s=i=0}}return t.x=this.a*s+this.x0,t.y=this.a*i+this.y0,t},inverse:function(t){t.x-=this.x0,t.y-=this.y0;var s,i,a,h,e,n,r,o,l,M,c=t.x/this.a,u=t.y/this.a;if(this.sphere){var f=0,m=0,p=Math.sqrt(c*c+u*u);if(1<(i=.5*p))return null;switch(i=2*Math.asin(i),this.mode!==this.OBLIQ&&this.mode!==this.EQUIT||(m=Math.sin(i),f=Math.cos(i)),this.mode){case this.EQUIT:i=Math.abs(p)<=D?0:Math.asin(u*m/p),c*=m,u=f*p;break;case this.OBLIQ:i=Math.abs(p)<=D?this.lat0:Math.asin(f*this.sinph0+u*m*this.cosph0/p),c*=m*this.cosph0,u=(f-Math.sin(i)*this.sinph0)*p;break;case this.N_POLE:u=-u,i=z-i;break;case this.S_POLE:i-=z}s=0!==u||this.mode!==this.EQUIT&&this.mode!==this.OBLIQ?Math.atan2(c,u):0}else{if(r=0,this.mode===this.OBLIQ||this.mode===this.EQUIT){if(c/=this.dd,u*=this.dd,(n=Math.sqrt(c*c+u*u))<D)return t.x=this.long0,t.y=this.lat0,t;h=2*Math.asin(.5*n/this.rq),a=Math.cos(h),c*=h=Math.sin(h),u=this.mode===this.OBLIQ?(r=a*this.sinb1+u*h*this.cosb1/n,e=this.qp*r,n*this.cosb1*a-u*this.sinb1*h):(r=u*h/n,e=this.qp*r,n*a)}else if(this.mode===this.N_POLE||this.mode===this.S_POLE){if(this.mode===this.N_POLE&&(u=-u),!(e=c*c+u*u))return t.x=this.long0,t.y=this.lat0,t;r=1-e/this.qp,this.mode===this.S_POLE&&(r=-r)}s=Math.atan2(c,u),o=Math.asin(r),l=this.apa,M=o+o,i=o+l[0]*Math.sin(M)+l[1]*Math.sin(M+M)+l[2]*Math.sin(M+M+M)}return t.x=nt(this.long0+s),t.y=i,t},names:["Lambert Azimuthal Equal Area","Lambert_Azimuthal_Equal_Area","laea"],S_POLE:1,N_POLE:2,EQUIT:3,OBLIQ:4},fs={init:function(){Math.abs(this.lat1+this.lat2)<D||(this.temp=this.b/this.a,this.es=1-Math.pow(this.temp,2),this.e3=Math.sqrt(this.es),this.sin_po=Math.sin(this.lat1),this.cos_po=Math.cos(this.lat1),this.t1=this.sin_po,this.con=this.sin_po,this.ms1=ht(this.e3,this.sin_po,this.cos_po),this.qs1=Vt(this.e3,this.sin_po,this.cos_po),this.sin_po=Math.sin(this.lat2),this.cos_po=Math.cos(this.lat2),this.t2=this.sin_po,this.ms2=ht(this.e3,this.sin_po,this.cos_po),this.qs2=Vt(this.e3,this.sin_po,this.cos_po),this.sin_po=Math.sin(this.lat0),this.cos_po=Math.cos(this.lat0),this.t3=this.sin_po,this.qs0=Vt(this.e3,this.sin_po,this.cos_po),Math.abs(this.lat1-this.lat2)>D?this.ns0=(this.ms1*this.ms1-this.ms2*this.ms2)/(this.qs2-this.qs1):this.ns0=this.con,this.c=this.ms1*this.ms1+this.ns0*this.qs1,this.rh=this.a*Math.sqrt(this.c-this.ns0*this.qs0)/this.ns0)},forward:function(t){var s=t.x,i=t.y;this.sin_phi=Math.sin(i),this.cos_phi=Math.cos(i);var a=Vt(this.e3,this.sin_phi,this.cos_phi),h=this.a*Math.sqrt(this.c-this.ns0*a)/this.ns0,e=this.ns0*nt(s-this.long0),n=h*Math.sin(e)+this.x0,r=this.rh-h*Math.cos(e)+this.y0;return t.x=n,t.y=r,t},inverse:function(t){var s,i,a,h,e,n;return t.x-=this.x0,t.y=this.rh-t.y+this.y0,a=0<=this.ns0?(s=Math.sqrt(t.x*t.x+t.y*t.y),1):(s=-Math.sqrt(t.x*t.x+t.y*t.y),-1),(h=0)!==s&&(h=Math.atan2(a*t.x,a*t.y)),a=s*this.ns0/this.a,n=this.sphere?Math.asin((this.c-a*a)/(2*this.ns0)):(i=(this.c-a*a)/this.ns0,this.phi1z(this.e3,i)),e=nt(h/this.ns0+this.long0),t.x=e,t.y=n,t},names:["Albers_Conic_Equal_Area","Albers","aea"],phi1z:function(t,s){var i,a,h,e,n=Zt(.5*s);if(t<D)return n;for(var r=t*t,o=1;o<=25;o++)if(n+=e=.5*(h=1-(a=t*(i=Math.sin(n)))*a)*h/Math.cos(n)*(s/(1-r)-i/h+.5/t*Math.log((1-a)/(1+a))),Math.abs(e)<=1e-7)return n;return null}},ms={init:function(){this.sin_p14=Math.sin(this.lat0),this.cos_p14=Math.cos(this.lat0),this.infinity_dist=1e3*this.a,this.rc=1},forward:function(t){var s,i,a=t.x,h=t.y,e=nt(a-this.long0),n=Math.sin(h),r=Math.cos(h),o=Math.cos(e),l=0<(s=this.sin_p14*n+this.cos_p14*r*o)||Math.abs(s)<=D?(i=this.x0+this.a*r*Math.sin(e)/s,this.y0+this.a*(this.cos_p14*n-this.sin_p14*r*o)/s):(i=this.x0+this.infinity_dist*r*Math.sin(e),this.y0+this.infinity_dist*(this.cos_p14*n-this.sin_p14*r*o));return t.x=i,t.y=l,t},inverse:function(t){var s,i,a,h,e,n;return t.x=(t.x-this.x0)/this.a,t.y=(t.y-this.y0)/this.a,t.x/=this.k0,t.y/=this.k0,e=(s=Math.sqrt(t.x*t.x+t.y*t.y))?(h=Math.atan2(s,this.rc),i=Math.sin(h),a=Math.cos(h),n=Zt(a*this.sin_p14+t.y*i*this.cos_p14/s),e=Math.atan2(t.x*i,s*this.cos_p14*a-t.y*this.sin_p14*i),nt(this.long0+e)):(n=this.phic0,0),t.x=e,t.y=n,t},names:["gnom"]},ps={init:function(){this.sphere||(this.k0=ht(this.e,Math.sin(this.lat_ts),Math.cos(this.lat_ts)))},forward:function(t){var s,i,a,h=t.x,e=t.y,n=nt(h-this.long0);return a=this.sphere?(i=this.x0+this.a*n*Math.cos(this.lat_ts),this.y0+this.a*Math.sin(e)/Math.cos(this.lat_ts)):(s=Vt(this.e,Math.sin(e)),i=this.x0+this.a*this.k0*n,this.y0+this.a*s*.5/this.k0),t.x=i,t.y=a,t},inverse:function(t){var s,i;return t.x-=this.x0,t.y-=this.y0,this.sphere?(s=nt(this.long0+t.x/this.a/Math.cos(this.lat_ts)),i=Math.asin(t.y/this.a*Math.cos(this.lat_ts))):(i=function(t,s){var i=1-(1-t*t)/(2*t)*Math.log((1-t)/(1+t));if(Math.abs(Math.abs(s)-i)<1e-6)return s<0?-1*z:z;for(var a,h,e,n,r=Math.asin(.5*s),o=0;o<30;o++)if(h=Math.sin(r),e=Math.cos(r),n=t*h,r+=a=Math.pow(1-n*n,2)/(2*e)*(s/(1-t*t)-h/(1-n*n)+.5/t*Math.log((1-n)/(1+n))),Math.abs(a)<=1e-10)return r;return NaN}(this.e,2*t.y*this.k0/this.a),s=nt(this.long0+t.x/(this.a*this.k0))),t.x=s,t.y=i,t},names:["cea"]},ds={init:function(){this.x0=this.x0||0,this.y0=this.y0||0,this.lat0=this.lat0||0,this.long0=this.long0||0,this.lat_ts=this.lat_ts||0,this.title=this.title||"Equidistant Cylindrical (Plate Carre)",this.rc=Math.cos(this.lat_ts)},forward:function(t){var s=t.x,i=t.y,a=nt(s-this.long0),h=Jt(i-this.lat0);return t.x=this.x0+this.a*a*this.rc,t.y=this.y0+this.a*h,t},inverse:function(t){var s=t.x,i=t.y;return t.x=nt(this.long0+(s-this.x0)/(this.a*this.rc)),t.y=Jt(this.lat0+(i-this.y0)/this.a),t},names:["Equirectangular","Equidistant_Cylindrical","eqc"]},ys={init:function(){this.temp=this.b/this.a,this.es=1-Math.pow(this.temp,2),this.e=Math.sqrt(this.es),this.e0=Ft(this.es),this.e1=Qt(this.es),this.e2=Wt(this.es),this.e3=Xt(this.es),this.ml0=this.a*Ut(this.e0,this.e1,this.e2,this.e3,this.lat0)},forward:function(t){var s,i,a,h=t.x,e=t.y,n=nt(h-this.long0),r=n*Math.sin(e);return a=this.sphere?Math.abs(e)<=D?(i=this.a*n,-1*this.a*this.lat0):(i=this.a*Math.sin(r)/Math.tan(e),this.a*(Jt(e-this.lat0)+(1-Math.cos(r))/Math.tan(e))):Math.abs(e)<=D?(i=this.a*n,-1*this.ml0):(i=(s=Ht(this.a,this.e,Math.sin(e))/Math.tan(e))*Math.sin(r),this.a*Ut(this.e0,this.e1,this.e2,this.e3,e)-this.ml0+s*(1-Math.cos(r))),t.x=i+this.x0,t.y=a+this.y0,t},inverse:function(t){var s,i,a,h,e,n,r,o,l=t.x-this.x0,M=t.y-this.y0;if(this.sphere)if(Math.abs(M+this.a*this.lat0)<=D)s=nt(l/this.a+this.long0),i=0;else{for(var c,u=this.lat0+M/this.a,f=l*l/this.a/this.a+u*u,m=u,p=20;p;--p)if(m+=a=-1*(u*(m*(c=Math.tan(m))+1)-m-.5*(m*m+f)*c)/((m-u)/c-1),Math.abs(a)<=D){i=m;break}s=nt(this.long0+Math.asin(l*Math.tan(m)/this.a)/Math.sin(i))}else if(Math.abs(M+this.ml0)<=D)i=0,s=nt(this.long0+l/this.a);else{for(u=(this.ml0+M)/this.a,f=l*l/this.a/this.a+u*u,m=u,p=20;p;--p)if(o=this.e*Math.sin(m),h=Math.sqrt(1-o*o)*Math.tan(m),e=this.a*Ut(this.e0,this.e1,this.e2,this.e3,m),n=this.e0-2*this.e1*Math.cos(2*m)+4*this.e2*Math.cos(4*m)-6*this.e3*Math.cos(6*m),m-=a=(u*(h*(r=e/this.a)+1)-r-.5*h*(r*r+f))/(this.es*Math.sin(2*m)*(r*r+f-2*u*r)/(4*h)+(u-r)*(h*n-2/Math.sin(2*m))-n),Math.abs(a)<=D){i=m;break}h=Math.sqrt(1-this.es*Math.pow(Math.sin(i),2))*Math.tan(i),s=nt(this.long0+Math.asin(l*h/this.a)/Math.sin(i))}return t.x=s,t.y=i,t},names:["Polyconic","poly"]},_s={init:function(){this.A=[],this.A[1]=.6399175073,this.A[2]=-.1358797613,this.A[3]=.063294409,this.A[4]=-.02526853,this.A[5]=.0117879,this.A[6]=-.0055161,this.A[7]=.0026906,this.A[8]=-.001333,this.A[9]=67e-5,this.A[10]=-34e-5,this.B_re=[],this.B_im=[],this.B_re[1]=.7557853228,this.B_im[1]=0,this.B_re[2]=.249204646,this.B_im[2]=.003371507,this.B_re[3]=-.001541739,this.B_im[3]=.04105856,this.B_re[4]=-.10162907,this.B_im[4]=.01727609,this.B_re[5]=-.26623489,this.B_im[5]=-.36249218,this.B_re[6]=-.6870983,this.B_im[6]=-1.1651967,this.C_re=[],this.C_im=[],this.C_re[1]=1.3231270439,this.C_im[1]=0,this.C_re[2]=-.577245789,this.C_im[2]=-.007809598,this.C_re[3]=.508307513,this.C_im[3]=-.112208952,this.C_re[4]=-.15094762,this.C_im[4]=.18200602,this.C_re[5]=1.01418179,this.C_im[5]=1.64497696,this.C_re[6]=1.9660549,this.C_im[6]=2.5127645,this.D=[],this.D[1]=1.5627014243,this.D[2]=.5185406398,this.D[3]=-.03333098,this.D[4]=-.1052906,this.D[5]=-.0368594,this.D[6]=.007317,this.D[7]=.0122,this.D[8]=.00394,this.D[9]=-.0013},forward:function(t){for(var s=t.x,i=t.y-this.lat0,a=s-this.long0,h=i/j*1e-5,e=a,n=1,r=0,o=1;o<=10;o++)n*=h,r+=this.A[o]*n;var l,M=r,c=e,u=1,f=0,m=0,p=0;for(o=1;o<=6;o++)l=f*M+u*c,u=u*M-f*c,f=l,m=m+this.B_re[o]*u-this.B_im[o]*f,p=p+this.B_im[o]*u+this.B_re[o]*f;return t.x=p*this.a+this.x0,t.y=m*this.a+this.y0,t},inverse:function(t){var s,i=t.x,a=t.y,h=i-this.x0,e=(a-this.y0)/this.a,n=h/this.a,r=1,o=0,l=0,M=0;for(y=1;y<=6;y++)s=o*e+r*n,r=r*e-o*n,o=s,l=l+this.C_re[y]*r-this.C_im[y]*o,M=M+this.C_im[y]*r+this.C_re[y]*o;for(var c=0;c<this.iterations;c++){for(var u,f=l,m=M,p=e,d=n,y=2;y<=6;y++)u=m*l+f*M,f=f*l-m*M,m=u,p+=(y-1)*(this.B_re[y]*f-this.B_im[y]*m),d+=(y-1)*(this.B_im[y]*f+this.B_re[y]*m);f=1,m=0;var _=this.B_re[1],x=this.B_im[1];for(y=2;y<=6;y++)u=m*l+f*M,f=f*l-m*M,m=u,_+=y*(this.B_re[y]*f-this.B_im[y]*m),x+=y*(this.B_im[y]*f+this.B_re[y]*m);var g=_*_+x*x,l=(p*_+d*x)/g,M=(d*_-p*x)/g}var b=l,v=M,w=1,C=0;for(y=1;y<=9;y++)w*=b,C+=this.D[y]*w;var P=this.lat0+C*j*1e5,S=this.long0+v;return t.x=S,t.y=P,t},names:["New_Zealand_Map_Grid","nzmg"]},xs={init:function(){},forward:function(t){var s=t.x,i=t.y,a=nt(s-this.long0),h=this.x0+this.a*a,e=this.y0+this.a*Math.log(Math.tan(Math.PI/4+i/2.5))*1.25;return t.x=h,t.y=e,t},inverse:function(t){t.x-=this.x0,t.y-=this.y0;var s=nt(this.long0+t.x/this.a),i=2.5*(Math.atan(Math.exp(.8*t.y/this.a))-Math.PI/4);return t.x=s,t.y=i,t},names:["Miller_Cylindrical","mill"]},gs={init:function(){this.sphere?(this.n=1,this.m=0,this.es=0,this.C_y=Math.sqrt((this.m+1)/this.n),this.C_x=this.C_y/(this.m+1)):this.en=At(this.es)},forward:function(t){var s=t.x,i=t.y,s=nt(s-this.long0);if(this.sphere){if(this.m)for(var a=this.n*Math.sin(i),h=20;h;--h){var e=(this.m*i+Math.sin(i)-a)/(this.m+Math.cos(i));if(i-=e,Math.abs(e)<D)break}else i=1!==this.n?Math.asin(this.n*Math.sin(i)):i;l=this.a*this.C_x*s*(this.m+Math.cos(i)),o=this.a*this.C_y*i}else var n=Math.sin(i),r=Math.cos(i),o=this.a*Gt(i,n,r,this.en),l=this.a*s*r/Math.sqrt(1-this.es*n*n);return t.x=l,t.y=o,t},inverse:function(t){var s,i,a,h;return t.x-=this.x0,a=t.x/this.a,t.y-=this.y0,s=t.y/this.a,this.sphere?(s/=this.C_y,a/=this.C_x*(this.m+Math.cos(s)),this.m?s=Zt((this.m*s+Math.sin(s))/this.n):1!==this.n&&(s=Zt(Math.sin(s)/this.n)),a=nt(a+this.long0),s=Jt(s)):(s=jt(t.y/this.a,this.es,this.en),(h=Math.abs(s))<z?(h=Math.sin(s),i=this.long0+t.x*Math.sqrt(1-this.es*h*h)/(this.a*Math.cos(s)),a=nt(i)):h-D<z&&(a=this.long0)),t.x=a,t.y=s,t},names:["Sinusoidal","sinu"]},bs={init:function(){},forward:function(t){for(var s=t.x,i=t.y,a=nt(s-this.long0),h=i,e=Math.PI*Math.sin(i);;){var n=-(h+Math.sin(h)-e)/(1+Math.cos(h));if(h+=n,Math.abs(n)<D)break}h/=2,Math.PI/2-Math.abs(i)<D&&(a=0);var r=.900316316158*this.a*a*Math.cos(h)+this.x0,o=1.4142135623731*this.a*Math.sin(h)+this.y0;return t.x=r,t.y=o,t},inverse:function(t){var s,i;t.x-=this.x0,t.y-=this.y0,i=t.y/(1.4142135623731*this.a),.999999999999<Math.abs(i)&&(i=.999999999999),s=Math.asin(i);var a=nt(this.long0+t.x/(.900316316158*this.a*Math.cos(s)));a<-Math.PI&&(a=-Math.PI),a>Math.PI&&(a=Math.PI),i=(2*s+Math.sin(2*s))/Math.PI,1<Math.abs(i)&&(i=1);var h=Math.asin(i);return t.x=a,t.y=h,t},names:["Mollweide","moll"]},vs={init:function(){Math.abs(this.lat1+this.lat2)<D||(this.lat2=this.lat2||this.lat1,this.temp=this.b/this.a,this.es=1-Math.pow(this.temp,2),this.e=Math.sqrt(this.es),this.e0=Ft(this.es),this.e1=Qt(this.es),this.e2=Wt(this.es),this.e3=Xt(this.es),this.sinphi=Math.sin(this.lat1),this.cosphi=Math.cos(this.lat1),this.ms1=ht(this.e,this.sinphi,this.cosphi),this.ml1=Ut(this.e0,this.e1,this.e2,this.e3,this.lat1),Math.abs(this.lat1-this.lat2)<D?this.ns=this.sinphi:(this.sinphi=Math.sin(this.lat2),this.cosphi=Math.cos(this.lat2),this.ms2=ht(this.e,this.sinphi,this.cosphi),this.ml2=Ut(this.e0,this.e1,this.e2,this.e3,this.lat2),this.ns=(this.ms1-this.ms2)/(this.ml2-this.ml1)),this.g=this.ml1+this.ms1/this.ns,this.ml0=Ut(this.e0,this.e1,this.e2,this.e3,this.lat0),this.rh=this.a*(this.g-this.ml0))},forward:function(t){var s,i,a=t.x,h=t.y;i=this.sphere?this.a*(this.g-h):(s=Ut(this.e0,this.e1,this.e2,this.e3,h),this.a*(this.g-s));var e=this.ns*nt(a-this.long0),n=this.x0+i*Math.sin(e),r=this.y0+this.rh-i*Math.cos(e);return t.x=n,t.y=r,t},inverse:function(t){var s,i;t.x-=this.x0,t.y=this.rh-t.y+this.y0,s=0<=this.ns?(i=Math.sqrt(t.x*t.x+t.y*t.y),1):(i=-Math.sqrt(t.x*t.x+t.y*t.y),-1);var a=0;if(0!==i&&(a=Math.atan2(s*t.x,s*t.y)),this.sphere)return n=nt(this.long0+a/this.ns),e=Jt(this.g-i/this.a),t.x=n,t.y=e,t;var h=this.g-i/this.a,e=Kt(h,this.e0,this.e1,this.e2,this.e3),n=nt(this.long0+a/this.ns);return t.x=n,t.y=e,t},names:["Equidistant_Conic","eqdc"]},ws={init:function(){this.R=this.a},forward:function(t){var s,i=t.x,a=t.y,h=nt(i-this.long0);Math.abs(a)<=D&&(s=this.x0+this.R*h,d=this.y0);var e=Zt(2*Math.abs(a/Math.PI));(Math.abs(h)<=D||Math.abs(Math.abs(a)-z)<=D)&&(s=this.x0,d=0<=a?this.y0+Math.PI*this.R*Math.tan(.5*e):this.y0+Math.PI*this.R*-Math.tan(.5*e));var n=.5*Math.abs(Math.PI/h-h/Math.PI),r=n*n,o=Math.sin(e),l=Math.cos(e),M=l/(o+l-1),c=M*M,u=M*(2/o-1),f=u*u,m=Math.PI*this.R*(n*(M-f)+Math.sqrt(r*(M-f)*(M-f)-(f+r)*(c-f)))/(f+r);h<0&&(m=-m),s=this.x0+m;var p=r+M,m=Math.PI*this.R*(u*p-n*Math.sqrt((f+r)*(1+r)-p*p))/(f+r),d=0<=a?this.y0+m:this.y0-m;return t.x=s,t.y=d,t},inverse:function(t){var s,i,a,h,e,n,r,o,l,M,c,u;return t.x-=this.x0,t.y-=this.y0,c=Math.PI*this.R,e=(a=t.x/c)*a+(h=t.y/c)*h,c=3*(h*h/(o=-2*(n=-Math.abs(h)*(1+e))+1+2*h*h+e*e)+(2*(r=n-2*h*h+a*a)*r*r/o/o/o-9*n*r/o/o)/27)/(l=(n-r*r/3/o)/o)/(M=2*Math.sqrt(-l/3)),1<Math.abs(c)&&(c=0<=c?1:-1),u=Math.acos(c)/3,i=0<=t.y?(-M*Math.cos(u+Math.PI/3)-r/3/o)*Math.PI:-(-M*Math.cos(u+Math.PI/3)-r/3/o)*Math.PI,s=Math.abs(a)<D?this.long0:nt(this.long0+Math.PI*(e-1+Math.sqrt(1+2*(a*a-h*h)+e*e))/2/a),t.x=s,t.y=i,t},names:["Van_der_Grinten_I","VanDerGrinten","vandg"]},Cs={init:function(){this.sin_p12=Math.sin(this.lat0),this.cos_p12=Math.cos(this.lat0)},forward:function(t){var s,i,a,h,e,n,r,o,l,M,c,u,f,m,p,d,y,_,x,g,b,v,w=t.x,C=t.y,P=Math.sin(t.y),S=Math.cos(t.y),N=nt(w-this.long0);return this.sphere?Math.abs(this.sin_p12-1)<=D?(t.x=this.x0+this.a*(z-C)*Math.sin(N),t.y=this.y0-this.a*(z-C)*Math.cos(N)):Math.abs(this.sin_p12+1)<=D?(t.x=this.x0+this.a*(z+C)*Math.sin(N),t.y=this.y0+this.a*(z+C)*Math.cos(N)):(_=this.sin_p12*P+this.cos_p12*S*Math.cos(N),y=(d=Math.acos(_))?d/Math.sin(d):1,t.x=this.x0+this.a*y*S*Math.sin(N),t.y=this.y0+this.a*y*(this.cos_p12*P-this.sin_p12*S*Math.cos(N))):(s=Ft(this.es),i=Qt(this.es),a=Wt(this.es),h=Xt(this.es),Math.abs(this.sin_p12-1)<=D?(e=this.a*Ut(s,i,a,h,z),n=this.a*Ut(s,i,a,h,C),t.x=this.x0+(e-n)*Math.sin(N),t.y=this.y0-(e-n)*Math.cos(N)):Math.abs(this.sin_p12+1)<=D?(e=this.a*Ut(s,i,a,h,z),n=this.a*Ut(s,i,a,h,C),t.x=this.x0+(e+n)*Math.sin(N),t.y=this.y0+(e+n)*Math.cos(N)):(r=P/S,o=Ht(this.a,this.e,this.sin_p12),l=Ht(this.a,this.e,P),M=Math.atan((1-this.es)*r+this.es*o*this.sin_p12/(l*S)),x=0===(c=Math.atan2(Math.sin(N),this.cos_p12*Math.tan(M)-this.sin_p12*Math.cos(N)))?Math.asin(this.cos_p12*Math.sin(M)-this.sin_p12*Math.cos(M)):Math.abs(Math.abs(c)-Math.PI)<=D?-Math.asin(this.cos_p12*Math.sin(M)-this.sin_p12*Math.cos(M)):Math.asin(Math.sin(N)*Math.cos(M)/Math.sin(c)),u=this.e*this.sin_p12/Math.sqrt(1-this.es),d=o*x*(1-(g=x*x)*(p=(f=this.e*this.cos_p12*Math.cos(c)/Math.sqrt(1-this.es))*f)*(1-p)/6+(b=g*x)/8*(m=u*f)*(1-2*p)+(v=b*x)/120*(p*(4-7*p)-3*u*u*(1-7*p))-v*x/48*m),t.x=this.x0+d*Math.sin(c),t.y=this.y0+d*Math.cos(c))),t},inverse:function(t){var s,i,a,h,e,n,r,o,l,M,c,u,f,m,p,d,y,_,x,g,b,v,w;if(t.x-=this.x0,t.y-=this.y0,this.sphere){if((s=Math.sqrt(t.x*t.x+t.y*t.y))>2*z*this.a)return;return i=s/this.a,a=Math.sin(i),h=Math.cos(i),e=this.long0,Math.abs(s)<=D?n=this.lat0:(n=Zt(h*this.sin_p12+t.y*a*this.cos_p12/s),r=Math.abs(this.lat0)-z,e=nt(Math.abs(r)<=D?0<=this.lat0?this.long0+Math.atan2(t.x,-t.y):this.long0-Math.atan2(-t.x,t.y):this.long0+Math.atan2(t.x*a,s*this.cos_p12*h-t.y*this.sin_p12*a))),t.x=e,t.y=n,t}return o=Ft(this.es),l=Qt(this.es),M=Wt(this.es),c=Xt(this.es),Math.abs(this.sin_p12-1)<=D?(u=this.a*Ut(o,l,M,c,z),s=Math.sqrt(t.x*t.x+t.y*t.y),n=Kt((u-s)/this.a,o,l,M,c),e=nt(this.long0+Math.atan2(t.x,-1*t.y))):Math.abs(this.sin_p12+1)<=D?(u=this.a*Ut(o,l,M,c,z),s=Math.sqrt(t.x*t.x+t.y*t.y),n=Kt((s-u)/this.a,o,l,M,c),e=nt(this.long0+Math.atan2(t.x,t.y))):(s=Math.sqrt(t.x*t.x+t.y*t.y),p=Math.atan2(t.x,t.y),f=Ht(this.a,this.e,this.sin_p12),d=Math.cos(p),_=-(y=this.e*this.cos_p12*d)*y/(1-this.es),x=3*this.es*(1-_)*this.sin_p12*this.cos_p12*d/(1-this.es),v=1-_*(b=(g=s/f)-_*(1+_)*Math.pow(g,3)/6-x*(1+3*_)*Math.pow(g,4)/24)*b/2-g*b*b*b/6,m=Math.asin(this.sin_p12*Math.cos(b)+this.cos_p12*Math.sin(b)*d),e=nt(this.long0+Math.asin(Math.sin(p)*Math.sin(b)/Math.cos(m))),w=Math.sin(m),n=Math.atan2((w-this.es*v*this.sin_p12)*Math.tan(m),w*(1-this.es))),t.x=e,t.y=n,t},names:["Azimuthal_Equidistant","aeqd"]},Ps={init:function(){this.sin_p14=Math.sin(this.lat0),this.cos_p14=Math.cos(this.lat0)},forward:function(t){var s,i,a,h=t.x,e=t.y,n=nt(h-this.long0),r=Math.sin(e),o=Math.cos(e),l=Math.cos(n);return(0<(s=this.sin_p14*r+this.cos_p14*o*l)||Math.abs(s)<=D)&&(i=this.a*o*Math.sin(n),a=this.y0+this.a*(this.cos_p14*r-this.sin_p14*o*l)),t.x=i,t.y=a,t},inverse:function(t){var s,i,a,h,e,n,r;return t.x-=this.x0,t.y-=this.y0,s=Math.sqrt(t.x*t.x+t.y*t.y),i=Zt(s/this.a),a=Math.sin(i),h=Math.cos(i),n=this.long0,Math.abs(s)<=D?r=this.lat0:(r=Zt(h*this.sin_p14+t.y*a*this.cos_p14/s),e=Math.abs(this.lat0)-z,n=Math.abs(e)<=D?nt(0<=this.lat0?this.long0+Math.atan2(t.x,-t.y):this.long0-Math.atan2(-t.x,t.y)):nt(this.long0+Math.atan2(t.x*a,s*this.cos_p14*h-t.y*this.sin_p14*a))),t.x=n,t.y=r,t},names:["ortho"]},Ss=1,Ns=2,ks=3,Es=4,qs=5,Is=6,Os=1,As=2,Gs=3,js=4,zs={init:function(){this.x0=this.x0||0,this.y0=this.y0||0,this.lat0=this.lat0||0,this.long0=this.long0||0,this.lat_ts=this.lat_ts||0,this.title=this.title||"Quadrilateralized Spherical Cube",this.lat0>=z-U/2?this.face=qs:this.lat0<=-(z-U/2)?this.face=Is:Math.abs(this.long0)<=U?this.face=Ss:Math.abs(this.long0)<=z+U?this.face=0<this.long0?Ns:Es:this.face=ks,0!==this.es&&(this.one_minus_f=1-(this.a-this.b)/this.a,this.one_minus_f_squared=this.one_minus_f*this.one_minus_f)},forward:function(t){var s,i,a,h,e,n,r,o,l,M,c,u,f={x:0,y:0},m={value:0};return t.x-=this.long0,s=0!==this.es?Math.atan(this.one_minus_f_squared*Math.tan(t.y)):t.y,i=t.x,this.face===qs?(h=z-s,a=U<=i&&i<=z+U?(m.value=Os,i-z):z+U<i||i<=-(z+U)?(m.value=As,0<i?i-Q:i+Q):-(z+U)<i&&i<=-U?(m.value=Gs,i+z):(m.value=js,i)):this.face===Is?(h=z+s,a=U<=i&&i<=z+U?(m.value=Os,z-i):i<U&&-U<=i?(m.value=As,-i):i<-U&&-(z+U)<=i?(m.value=Gs,-i-z):(m.value=js,0<i?Q-i:-i-Q)):(this.face===Ns?i=S(i,+z):this.face===ks?i=S(i,+Q):this.face===Es&&(i=S(i,-z)),M=Math.sin(s),c=Math.cos(s),u=Math.sin(i),r=c*Math.cos(i),o=c*u,l=M,this.face===Ss?a=P(h=Math.acos(r),l,o,m):this.face===Ns?a=P(h=Math.acos(o),l,-r,m):this.face===ks?a=P(h=Math.acos(-r),l,-o,m):this.face===Es?a=P(h=Math.acos(-o),l,r,m):(h=a=0,m.value=Os)),n=Math.atan(12/Q*(a+Math.acos(Math.sin(a)*Math.cos(U))-z)),e=Math.sqrt((1-Math.cos(h))/(Math.cos(n)*Math.cos(n))/(1-Math.cos(Math.atan(1/Math.cos(a))))),m.value===As?n+=z:m.value===Gs?n+=Q:m.value===js&&(n+=1.5*Q),f.x=e*Math.cos(n),f.y=e*Math.sin(n),f.x=f.x*this.a+this.x0,f.y=f.y*this.a+this.y0,t.x=f.x,t.y=f.y,t},inverse:function(t){var s,i,a,h,e,n,r,o,l,M,c,u,f,m,p,d={lam:0,phi:0},y={value:0};return t.x=(t.x-this.x0)/this.a,t.y=(t.y-this.y0)/this.a,i=Math.atan(Math.sqrt(t.x*t.x+t.y*t.y)),s=Math.atan2(t.y,t.x),0<=t.x&&t.x>=Math.abs(t.y)?y.value=Os:0<=t.y&&t.y>=Math.abs(t.x)?(y.value=As,s-=z):t.x<0&&-t.x>=Math.abs(t.y)?(y.value=Gs,s=s<0?s+Q:s-Q):(y.value=js,s+=z),c=Q/12*Math.tan(s),e=Math.sin(c)/(Math.cos(c)-1/Math.sqrt(2)),n=Math.atan(e),(r=1-(a=Math.cos(s))*a*(h=Math.tan(i))*h*(1-Math.cos(Math.atan(1/Math.cos(n)))))<-1?r=-1:1<r&&(r=1),this.face===qs?(o=Math.acos(r),d.phi=z-o,y.value===Os?d.lam=n+z:y.value===As?d.lam=n<0?n+Q:n-Q:y.value===Gs?d.lam=n-z:d.lam=n):this.face===Is?(o=Math.acos(r),d.phi=o-z,y.value===Os?d.lam=z-n:y.value===As?d.lam=-n:y.value===Gs?d.lam=-n-z:d.lam=n<0?-n-Q:Q-n):(c=(l=r)*l,u=1<=(c+=(M=1<=c?0:Math.sqrt(1-c)*Math.sin(n))*M)?0:Math.sqrt(1-c),y.value===As?(c=u,u=-M,M=c):y.value===Gs?(u=-u,M=-M):y.value===js&&(c=u,u=M,M=-c),this.face===Ns?(c=l,l=-u,u=c):this.face===ks?(l=-l,u=-u):this.face===Es&&(c=l,l=u,u=-c),d.phi=Math.acos(-M)-z,d.lam=Math.atan2(u,l),this.face===Ns?d.lam=S(d.lam,-z):this.face===ks?d.lam=S(d.lam,-Q):this.face===Es&&(d.lam=S(d.lam,+z))),0!==this.es&&(f=d.phi<0?1:0,m=Math.tan(d.phi),p=this.b/Math.sqrt(m*m+this.one_minus_f_squared),d.phi=Math.atan(Math.sqrt(this.a*this.a-p*p)/(this.one_minus_f*p)),f&&(d.phi=-d.phi)),d.lam+=this.long0,t.x=d.lam,t.y=d.phi,t},names:["Quadrilateralized Spherical Cube","Quadrilateralized_Spherical_Cube","qsc"]},Rs=[[1,22199e-21,-715515e-10,31103e-10],[.9986,-482243e-9,-24897e-9,-13309e-10],[.9954,-83103e-8,-448605e-10,-9.86701e-7],[.99,-.00135364,-59661e-9,36777e-10],[.9822,-.00167442,-449547e-11,-572411e-11],[.973,-.00214868,-903571e-10,1.8736e-8],[.96,-.00305085,-900761e-10,164917e-11],[.9427,-.00382792,-653386e-10,-26154e-10],[.9216,-.00467746,-10457e-8,481243e-11],[.8962,-.00536223,-323831e-10,-543432e-11],[.8679,-.00609363,-113898e-9,332484e-11],[.835,-.00698325,-640253e-10,9.34959e-7],[.7986,-.00755338,-500009e-10,9.35324e-7],[.7597,-.00798324,-35971e-9,-227626e-11],[.7186,-.00851367,-701149e-10,-86303e-10],[.6732,-.00986209,-199569e-9,191974e-10],[.6213,-.010418,883923e-10,624051e-11],[.5722,-.00906601,182e-6,624051e-11],[.5322,-.00677797,275608e-9,624051e-11]],Ls=[[-520417e-23,.0124,121431e-23,-845284e-16],[.062,.0124,-1.26793e-9,4.22642e-10],[.124,.0124,5.07171e-9,-1.60604e-9],[.186,.0123999,-1.90189e-8,6.00152e-9],[.248,.0124002,7.10039e-8,-2.24e-8],[.31,.0123992,-2.64997e-7,8.35986e-8],[.372,.0124029,9.88983e-7,-3.11994e-7],[.434,.0123893,-369093e-11,-4.35621e-7],[.4958,.0123198,-102252e-10,-3.45523e-7],[.5571,.0121916,-154081e-10,-5.82288e-7],[.6176,.0119938,-241424e-10,-5.25327e-7],[.6769,.011713,-320223e-10,-5.16405e-7],[.7346,.0113541,-397684e-10,-6.09052e-7],[.7903,.0109107,-489042e-10,-104739e-11],[.8435,.0103431,-64615e-9,-1.40374e-9],[.8936,.00969686,-64636e-9,-8547e-9],[.9394,.00840947,-192841e-9,-42106e-10],[.9761,.00616527,-256e-6,-42106e-10],[1,.00328947,-319159e-9,-42106e-10]],Ts=B/5,Ds=1/Ts,Bs={init:function(){this.x0=this.x0||0,this.y0=this.y0||0,this.long0=this.long0||0,this.es=0,this.title=this.title||"Robinson"},forward:function(t){var s=nt(t.x-this.long0),i=Math.abs(t.y),a=Math.floor(i*Ts);a<0?a=0:18<=a&&(a=17);var h={x:Yt(Rs[a],i=B*(i-Ds*a))*s,y:Yt(Ls[a],i)};return t.y<0&&(h.y=-h.y),h.x=h.x*this.a*.8487+this.x0,h.y=h.y*this.a*1.3523+this.y0,h},inverse:function(t){var a={x:(t.x-this.x0)/(.8487*this.a),y:Math.abs(t.y-this.y0)/(1.3523*this.a)};if(1<=a.y)a.x/=Rs[18][0],a.y=t.y<0?-z:z;else{var s=Math.floor(18*a.y);for(s<0?s=0:18<=s&&(s=17);;)if(Ls[s][0]>a.y)--s;else{if(!(Ls[s+1][0]<=a.y))break;++s}var h=Ls[s],i=function(t,s,i,a){for(var h=s;a;--a){var e=t(h);if(h-=e,Math.abs(e)<i)break}return h}(function(t){return(Yt(h,t)-a.y)/(i=t,(s=h)[1]+i*(2*s[2]+3*i*s[3]));var s,i},i=5*(a.y-h[0])/(Ls[s+1][0]-h[0]),D,100);a.x/=Yt(Rs[s],i),a.y=(5*s+i)*N,t.y<0&&(a.y=-a.y)}return a.x=nt(a.x+this.long0),a},names:["Robinson","robin"]},Us={init:function(){this.name="geocent"},forward:function(t){return M(t,this.es,this.a)},inverse:function(t){return c(t,this.es,this.a,this.b)},names:["Geocentric","geocentric","geocent","Geocent"]};return a.defaultDatum="WGS84",a.Proj=q,a.WGS84=new a.Proj("WGS84"),a.Point=C,a.toPoint=bt,a.defs=l,a.transform=f,a.mgrs=Ot,a.version="2.6.2",($t=a).Proj.projections.add(ss),$t.Proj.projections.add(is),$t.Proj.projections.add(as),$t.Proj.projections.add(es),$t.Proj.projections.add(ns),$t.Proj.projections.add(rs),$t.Proj.projections.add(os),$t.Proj.projections.add(ls),$t.Proj.projections.add(Ms),$t.Proj.projections.add(cs),$t.Proj.projections.add(us),$t.Proj.projections.add(fs),$t.Proj.projections.add(ms),$t.Proj.projections.add(ps),$t.Proj.projections.add(ds),$t.Proj.projections.add(ys),$t.Proj.projections.add(_s),$t.Proj.projections.add(xs),$t.Proj.projections.add(gs),$t.Proj.projections.add(bs),$t.Proj.projections.add(vs),$t.Proj.projections.add(ws),$t.Proj.projections.add(Cs),$t.Proj.projections.add(Ps),$t.Proj.projections.add(zs),$t.Proj.projections.add(Bs),$t.Proj.projections.add(Us),a});</script>
<script>(function (factory) {
	var L, proj4;
	if (typeof define === 'function' && define.amd) {
		// AMD
		define(['leaflet', 'proj4'], factory);
	} else if (typeof module === 'object' && typeof module.exports === "object") {
		// Node/CommonJS
		L = require('leaflet');
		proj4 = require('proj4');
		module.exports = factory(L, proj4);
	} else {
		// Browser globals
		if (typeof window.L === 'undefined' || typeof window.proj4 === 'undefined')
			throw 'Leaflet and proj4 must be loaded first';
		factory(window.L, window.proj4);
	}
}(function (L, proj4) {
	if (proj4.__esModule && proj4.default) {
		// If proj4 was bundled as an ES6 module, unwrap it to get
		// to the actual main proj4 object.
		// See discussion in https://github.com/kartena/Proj4Leaflet/pull/147
		proj4 = proj4.default;
	}
 
	L.Proj = {};

	L.Proj._isProj4Obj = function(a) {
		return (typeof a.inverse !== 'undefined' &&
			typeof a.forward !== 'undefined');
	};

	L.Proj.Projection = L.Class.extend({
		initialize: function(code, def, bounds) {
			var isP4 = L.Proj._isProj4Obj(code);
			this._proj = isP4 ? code : this._projFromCodeDef(code, def);
			this.bounds = isP4 ? def : bounds;
		},

		project: function (latlng) {
			var point = this._proj.forward([latlng.lng, latlng.lat]);
			return new L.Point(point[0], point[1]);
		},

		unproject: function (point, unbounded) {
			var point2 = this._proj.inverse([point.x, point.y]);
			return new L.LatLng(point2[1], point2[0], unbounded);
		},

		_projFromCodeDef: function(code, def) {
			if (def) {
				proj4.defs(code, def);
			} else if (proj4.defs[code] === undefined) {
				var urn = code.split(':');
				if (urn.length > 3) {
					code = urn[urn.length - 3] + ':' + urn[urn.length - 1];
				}
				if (proj4.defs[code] === undefined) {
					throw 'No projection definition for code ' + code;
				}
			}

			return proj4(code);
		}
	});

	L.Proj.CRS = L.Class.extend({
		includes: L.CRS,

		options: {
			transformation: new L.Transformation(1, 0, -1, 0)
		},

		initialize: function(a, b, c) {
			var code,
			    proj,
			    def,
			    options;

			if (L.Proj._isProj4Obj(a)) {
				proj = a;
				code = proj.srsCode;
				options = b || {};

				this.projection = new L.Proj.Projection(proj, options.bounds);
			} else {
				code = a;
				def = b;
				options = c || {};
				this.projection = new L.Proj.Projection(code, def, options.bounds);
			}

			L.Util.setOptions(this, options);
			this.code = code;
			this.transformation = this.options.transformation;

			if (this.options.origin) {
				this.transformation =
					new L.Transformation(1, -this.options.origin[0],
						-1, this.options.origin[1]);
			}

			if (this.options.scales) {
				this._scales = this.options.scales;
			} else if (this.options.resolutions) {
				this._scales = [];
				for (var i = this.options.resolutions.length - 1; i >= 0; i--) {
					if (this.options.resolutions[i]) {
						this._scales[i] = 1 / this.options.resolutions[i];
					}
				}
			}

			this.infinite = !this.options.bounds;

		},

		scale: function(zoom) {
			var iZoom = Math.floor(zoom),
				baseScale,
				nextScale,
				scaleDiff,
				zDiff;
			if (zoom === iZoom) {
				return this._scales[zoom];
			} else {
				// Non-integer zoom, interpolate
				baseScale = this._scales[iZoom];
				nextScale = this._scales[iZoom + 1];
				scaleDiff = nextScale - baseScale;
				zDiff = (zoom - iZoom);
				return baseScale + scaleDiff * zDiff;
			}
		},

		zoom: function(scale) {
			// Find closest number in this._scales, down
			var downScale = this._closestElement(this._scales, scale),
				downZoom = this._scales.indexOf(downScale),
				nextScale,
				nextZoom,
				scaleDiff;
			// Check if scale is downScale => return array index
			if (scale === downScale) {
				return downZoom;
			}
			if (downScale === undefined) {
				return -Infinity;
			}
			// Interpolate
			nextZoom = downZoom + 1;
			nextScale = this._scales[nextZoom];
			if (nextScale === undefined) {
				return Infinity;
			}
			scaleDiff = nextScale - downScale;
			return (scale - downScale) / scaleDiff + downZoom;
		},

		distance: L.CRS.Earth.distance,

		R: L.CRS.Earth.R,

		/* Get the closest lowest element in an array */
		_closestElement: function(array, element) {
			var low;
			for (var i = array.length; i--;) {
				if (array[i] <= element && (low === undefined || low < array[i])) {
					low = array[i];
				}
			}
			return low;
		}
	});

	L.Proj.GeoJSON = L.GeoJSON.extend({
		initialize: function(geojson, options) {
			this._callLevel = 0;
			L.GeoJSON.prototype.initialize.call(this, geojson, options);
		},

		addData: function(geojson) {
			var crs;

			if (geojson) {
				if (geojson.crs && geojson.crs.type === 'name') {
					crs = new L.Proj.CRS(geojson.crs.properties.name);
				} else if (geojson.crs && geojson.crs.type) {
					crs = new L.Proj.CRS(geojson.crs.type + ':' + geojson.crs.properties.code);
				}

				if (crs !== undefined) {
					this.options.coordsToLatLng = function(coords) {
						var point = L.point(coords[0], coords[1]);
						return crs.projection.unproject(point);
					};
				}
			}

			// Base class' addData might call us recursively, but
			// CRS shouldn't be cleared in that case, since CRS applies
			// to the whole GeoJSON, inluding sub-features.
			this._callLevel++;
			try {
				L.GeoJSON.prototype.addData.call(this, geojson);
			} finally {
				this._callLevel--;
				if (this._callLevel === 0) {
					delete this.options.coordsToLatLng;
				}
			}
		}
	});

	L.Proj.geoJson = function(geojson, options) {
		return new L.Proj.GeoJSON(geojson, options);
	};

	L.Proj.ImageOverlay = L.ImageOverlay.extend({
		initialize: function (url, bounds, options) {
			L.ImageOverlay.prototype.initialize.call(this, url, null, options);
			this._projectedBounds = bounds;
		},

		// Danger ahead: Overriding internal methods in Leaflet.
		// Decided to do this rather than making a copy of L.ImageOverlay
		// and doing very tiny modifications to it.
		// Future will tell if this was wise or not.
		_animateZoom: function (event) {
			var scale = this._map.getZoomScale(event.zoom);
			var northWest = L.point(this._projectedBounds.min.x, this._projectedBounds.max.y);
			var offset = this._projectedToNewLayerPoint(northWest, event.zoom, event.center);

			L.DomUtil.setTransform(this._image, offset, scale);
		},

		_reset: function () {
			var zoom = this._map.getZoom();
			var pixelOrigin = this._map.getPixelOrigin();
			var bounds = L.bounds(
				this._transform(this._projectedBounds.min, zoom)._subtract(pixelOrigin),
				this._transform(this._projectedBounds.max, zoom)._subtract(pixelOrigin)
			);
			var size = bounds.getSize();

			L.DomUtil.setPosition(this._image, bounds.min);
			this._image.style.width = size.x + 'px';
			this._image.style.height = size.y + 'px';
		},

		_projectedToNewLayerPoint: function (point, zoom, center) {
			var viewHalf = this._map.getSize()._divideBy(2);
			var newTopLeft = this._map.project(center, zoom)._subtract(viewHalf)._round();
			var topLeft = newTopLeft.add(this._map._getMapPanePos());

			return this._transform(point, zoom)._subtract(topLeft);
		},

		_transform: function (point, zoom) {
			var crs = this._map.options.crs;
			var transformation = crs.transformation;
			var scale = crs.scale(zoom);

			return transformation.transform(point, scale);
		}
	});

	L.Proj.imageOverlay = function (url, bounds, options) {
		return new L.Proj.ImageOverlay(url, bounds, options);
	};

	return L.Proj;
}));
</script>
<style type="text/css">.leaflet-tooltip.leaflet-tooltip-text-only,
.leaflet-tooltip.leaflet-tooltip-text-only:before,
.leaflet-tooltip.leaflet-tooltip-text-only:after {
background: none;
border: none;
box-shadow: none;
}
.leaflet-tooltip.leaflet-tooltip-text-only.leaflet-tooltip-left {
margin-left: 5px;
}
.leaflet-tooltip.leaflet-tooltip-text-only.leaflet-tooltip-right {
margin-left: -5px;
}
.leaflet-tooltip:after {
border-right: 6px solid transparent;

}
.leaflet-popup-pane .leaflet-popup-tip-container {

pointer-events: all;

cursor: pointer;
}

.leaflet-map-pane {
z-index: auto;
}
</style>
<script>(function(){function r(e,n,t){function o(i,f){if(!n[i]){if(!e[i]){var c="function"==typeof require&&require;if(!f&&c)return c(i,!0);if(u)return u(i,!0);var a=new Error("Cannot find module '"+i+"'");throw a.code="MODULE_NOT_FOUND",a}var p=n[i]={exports:{}};e[i][0].call(p.exports,function(r){var n=e[i][1][r];return o(n||r)},p,p.exports,r,e,n,t)}return n[i].exports}for(var u="function"==typeof require&&require,i=0;i<t.length;i++)o(t[i]);return o}return r})()({1:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = undefined;

var _util = require("./util");

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

var ClusterLayerStore = /*#__PURE__*/function () {
  function ClusterLayerStore(group) {
    _classCallCheck(this, ClusterLayerStore);

    this._layers = {};
    this._group = group;
  }

  _createClass(ClusterLayerStore, [{
    key: "add",
    value: function add(layer, id) {
      if (typeof id !== "undefined" && id !== null) {
        if (this._layers[id]) {
          this._group.removeLayer(this._layers[id]);
        }

        this._layers[id] = layer;
      }

      this._group.addLayer(layer);
    }
  }, {
    key: "remove",
    value: function remove(id) {
      if (typeof id === "undefined" || id === null) {
        return;
      }

      id = (0, _util.asArray)(id);

      for (var i = 0; i < id.length; i++) {
        if (this._layers[id[i]]) {
          this._group.removeLayer(this._layers[id[i]]);

          delete this._layers[id[i]];
        }
      }
    }
  }, {
    key: "clear",
    value: function clear() {
      this._layers = {};

      this._group.clearLayers();
    }
  }]);

  return ClusterLayerStore;
}();

exports["default"] = ClusterLayerStore;


},{"./util":17}],2:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

var ControlStore = /*#__PURE__*/function () {
  function ControlStore(map) {
    _classCallCheck(this, ControlStore);

    this._controlsNoId = [];
    this._controlsById = {};
    this._map = map;
  }

  _createClass(ControlStore, [{
    key: "add",
    value: function add(control, id, html) {
      if (typeof id !== "undefined" && id !== null) {
        if (this._controlsById[id]) {
          this._map.removeControl(this._controlsById[id]);
        }

        this._controlsById[id] = control;
      } else {
        this._controlsNoId.push(control);
      }

      this._map.addControl(control);
    }
  }, {
    key: "get",
    value: function get(id) {
      var control = null;

      if (this._controlsById[id]) {
        control = this._controlsById[id];
      }

      return control;
    }
  }, {
    key: "remove",
    value: function remove(id) {
      if (this._controlsById[id]) {
        var control = this._controlsById[id];

        this._map.removeControl(control);

        delete this._controlsById[id];
      }
    }
  }, {
    key: "clear",
    value: function clear() {
      for (var i = 0; i < this._controlsNoId.length; i++) {
        var control = this._controlsNoId[i];

        this._map.removeControl(control);
      }

      this._controlsNoId = [];

      for (var key in this._controlsById) {
        var _control = this._controlsById[key];

        this._map.removeControl(_control);
      }

      this._controlsById = {};
    }
  }]);

  return ControlStore;
}();

exports["default"] = ControlStore;


},{}],3:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.getCRS = getCRS;

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

var _proj4leaflet = require("./global/proj4leaflet");

var _proj4leaflet2 = _interopRequireDefault(_proj4leaflet);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

// Helper function to instanciate a ICRS instance.
function getCRS(crsOptions) {
  var crs = _leaflet2["default"].CRS.EPSG3857; // Default Spherical Mercator

  switch (crsOptions.crsClass) {
    case "L.CRS.EPSG3857":
      crs = _leaflet2["default"].CRS.EPSG3857;
      break;

    case "L.CRS.EPSG4326":
      crs = _leaflet2["default"].CRS.EPSG4326;
      break;

    case "L.CRS.EPSG3395":
      crs = _leaflet2["default"].CRS.EPSG3395;
      break;

    case "L.CRS.Simple":
      crs = _leaflet2["default"].CRS.Simple;
      break;

    case "L.Proj.CRS":
      if (crsOptions.options && crsOptions.options.bounds) {
        crsOptions.options.bounds = _leaflet2["default"].bounds(crsOptions.options.bounds);
      }

      if (crsOptions.options && crsOptions.options.transformation) {
        crsOptions.options.transformation = new _leaflet2["default"].Transformation(crsOptions.options.transformation[0], crsOptions.options.transformation[1], crsOptions.options.transformation[2], crsOptions.options.transformation[3]);
      }

      crs = new _proj4leaflet2["default"].CRS(crsOptions.code, crsOptions.proj4def, crsOptions.options);
      break;

    case "L.Proj.CRS.TMS":
      if (crsOptions.options && crsOptions.options.bounds) {
        crsOptions.options.bounds = _leaflet2["default"].bounds(crsOptions.options.bounds);
      }

      if (crsOptions.options && crsOptions.options.transformation) {
        crsOptions.options.transformation = _leaflet2["default"].Transformation(crsOptions.options.transformation[0], crsOptions.options.transformation[1], crsOptions.options.transformation[2], crsOptions.options.transformation[3]);
      } // L.Proj.CRS.TMS is deprecated as of Leaflet 1.x, fall back to L.Proj.CRS
      //crs = new Proj4Leaflet.CRS.TMS(crsOptions.code, crsOptions.proj4def, crsOptions.projectedBounds, crsOptions.options);


      crs = new _proj4leaflet2["default"].CRS(crsOptions.code, crsOptions.proj4def, crsOptions.options);
      break;
  }

  return crs;
}


},{"./global/leaflet":10,"./global/proj4leaflet":11}],4:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = undefined;

var _util = require("./util");

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

var DataFrame = /*#__PURE__*/function () {
  function DataFrame() {
    _classCallCheck(this, DataFrame);

    this.columns = [];
    this.colnames = [];
    this.colstrict = [];
    this.effectiveLength = 0;
    this.colindices = {};
  }

  _createClass(DataFrame, [{
    key: "_updateCachedProperties",
    value: function _updateCachedProperties() {
      var _this = this;

      this.effectiveLength = 0;
      this.colindices = {};
      this.columns.forEach(function (column, i) {
        _this.effectiveLength = Math.max(_this.effectiveLength, column.length);
        _this.colindices[_this.colnames[i]] = i;
      });
    }
  }, {
    key: "_colIndex",
    value: function _colIndex(colname) {
      var index = this.colindices[colname];
      if (typeof index === "undefined") return -1;
      return index;
    }
  }, {
    key: "col",
    value: function col(name, values, strict) {
      if (typeof name !== "string") throw new Error("Invalid column name \"" + name + "\"");

      var index = this._colIndex(name);

      if (arguments.length === 1) {
        if (index < 0) return null;else return (0, _util.recycle)(this.columns[index], this.effectiveLength);
      }

      if (index < 0) {
        index = this.colnames.length;
        this.colnames.push(name);
      }

      this.columns[index] = (0, _util.asArray)(values);
      this.colstrict[index] = !!strict; // TODO: Validate strictness (ensure lengths match up with other stricts)

      this._updateCachedProperties();

      return this;
    }
  }, {
    key: "cbind",
    value: function cbind(obj, strict) {
      var _this2 = this;

      Object.keys(obj).forEach(function (name) {
        var coldata = obj[name];

        _this2.col(name, coldata);
      });
      return this;
    }
  }, {
    key: "get",
    value: function get(row, col, missingOK) {
      var _this3 = this;

      if (row > this.effectiveLength) throw new Error("Row argument was out of bounds: " + row + " > " + this.effectiveLength);
      var colIndex = -1;

      if (typeof col === "undefined") {
        var rowData = {};
        this.colnames.forEach(function (name, i) {
          rowData[name] = _this3.columns[i][row % _this3.columns[i].length];
        });
        return rowData;
      } else if (typeof col === "string") {
        colIndex = this._colIndex(col);
      } else if (typeof col === "number") {
        colIndex = col;
      }

      if (colIndex < 0 || colIndex > this.columns.length) {
        if (missingOK) return void 0;else throw new Error("Unknown column index: " + col);
      }

      return this.columns[colIndex][row % this.columns[colIndex].length];
    }
  }, {
    key: "nrow",
    value: function nrow() {
      return this.effectiveLength;
    }
  }]);

  return DataFrame;
}();

exports["default"] = DataFrame;


},{"./util":17}],5:[function(require,module,exports){
"use strict";

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

// In RMarkdown's self-contained mode, we don't have a way to carry around the
// images that Leaflet needs but doesn't load into the page. Instead, we'll set
// data URIs for the default marker, and let any others be loaded via CDN.
if (typeof _leaflet2["default"].Icon.Default.imagePath === "undefined") {
  // if in a local file, support http
  switch (window.location.protocol) {
    case "http:":
      // don't force http site to be done with https
      _leaflet2["default"].Icon.Default.imagePath = "http://cdn.leafletjs.com/leaflet/v1.3.1/images/";
      break;

    default:
      // file
      // https
      // otherwise use https as it works on files and https
      _leaflet2["default"].Icon.Default.imagePath = "https://unpkg.com/leaflet@1.3.1/dist/images/";
      break;
  } // don't know how to make this dataURI work since
  //  will be appended to Defaul.imagePath above

  /*
  if (L.Browser.retina) {
    L.Icon.Default.prototype.options.iconUrl = "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAABSCAYAAAAWy4frAAAPiElEQVR42t1bCVCU5xkmbabtZJJOO+l0mhgT0yQe0WXZgz2570NB8I6J6UzaTBoORRFEruVGDhWUPRAQRFFREDnVxCtEBRb24DBNE3Waaatpkmluo4m+fd9v999olGVBDu3OPLj+//s+7/W93/f9//6/EwA4/T9g3AlFOUeeUGR2uMqzOyJk2R2x0qyOAmnmkS3SrCPrZJlHlsqzjypcs49OX1Jf//P7KhD885A0u10my2ovQscvybI6wEF8ivI7pFntAV6qkw9PWSBK1bEnZRltm2WZ7R8h4FbI0VG33GPgXXgCAra+A4EIn8KT4JH/FigoiJ/IIz6TZbVVKLLan5u0QESqlkckWW3p0sy2bxDAgZwO13TDytoB+NPe9+zild2DEFGuB7/NpzDodriF55o0o7XIRXXoNxMaiCSj9VU09C8EENxyj0C4thterh2EV+veuwOr6s7Dy3ssoO93k3llzxBE6PTgkXcMOF7EJ9KMtqjR9JFDQnNV9b+QqlqqEECQZ7TBgu1nYdXuIXgVneSwYtcgRFb1Q1iFGULLzRCsM90GOrZghxkiKvthec0grLpFlxCu6cKh1w6cHUSbctPhx8YlEElu4+NSVfNpBBACtpyGlbsGmBOElRhMBDofgk4GobOjQXC5CRZiUC/VDtn4qLrBJZ3A2cNg+nE4P31PgSDBbImq5UNJejMQFqi7cCicZ3iZBTAAQVoTBI4DKKCVGBDHH6nrBRlWxWr7sljVIhlTIDLVoRkS1eH/SNIPgzyzFRZV9NnG++LqQcyoGQLQgfFEIFYpcueAzc6SSiMOtTYgH9CXr+WpTbxRBeKlqn9UktZkRoACZ5PlO81YgfMM4RX9EKAxTSjCdvTjELPYW17dD8rsdiBfEBclSY2POxQIHnlIknroEAJk6U2wpMLISF/aNQShWAV/tWlSEIK2VqBNsr200gRyGmLokyS18cTdFtA7AnFNbcxAACGMrQtDLAjqBT+1cVJBNsk2+bBQ1wOcX5K0xs12A8GyzXRNafgeAYFb3mEkrBI4I/mWGUeNQI1lyp2PoO9j4aDKcH4Ebe0E8g3xgyylcc6wgbimNjSSoFtWK1sTqLRh2BM+SOgIfDGLJL8IG3ZZjUX/ViyvGYLFOwdZn/ljYI7yzsee4TjcsV/IR3FqQ+tdAxEnNSjFyQeBEK7pgRVodEnVIPhsNzqEYK0ZluFsRnq3YjH22KJyA6z4yTmSpZ5zlH8RTvWkt1CrB85PYUqjzx2BuG6sPyfeeAA8sjtwphhiCFSbwXub0S7ISPiOAZvO4h048xSfBM+cDpDieCZOggSz6JHdBv5FJ3CN6LPJR1QMgO9204h2aALgdDxzjlp4kw8YaHKyBSJJPigWb6wHQiRmbxkKL0QDXkhgD94YxGKsGskTQkvfxVnlIHBcBNfkegziwB3HAnHDuGynRXcp/utXZhrRHiWM5CPLjbdwHVDYAhFt3J8rTtoPbpktSDrE4INZ8iw12kUYEpPs4kozeOW0A3EQIovbYcfxITj798vwxbfX4Or1H8B46ROo7fwbvKY9bpNzy2hmiSOOyMrBEe2RT5x/7tjHxCFK2l/4YyBJ+95HQABmibKzEJvRs9RgF4FqE5MleGS3AumLN+6D4lYjfIeOD/e5eROg7sz7oEg7wHRk6Y3Yi/2MJwT7bCS75BvJBuGsSvqID1ggaHyeaAMeQERgyajBg3BG8SgxDAsvJFxUOcBkg7d0Ml3XjfuhCyvg6Ofix1+Al6qB6fpueotxsckFh5A92+QbydHw4vymGJxEG+rWiRL3goJWcSwvwbPECO5bDcMiRGNmchS4a1I9kP62DhOM9tPad4npEhaUdTPOsPJ+u7bJN85PpaqJ6YoT6xKcRIl1pQjwxIukxXhyIY57N1Swh7DyASbrm38MSHdRUStc+/4GjOUTV32acbhlNjNO6pWR7FPTk6xX3lGmK0ys0zrhn0Zhwh7wK3ibnVyg6we3LQa7WFQxyGSpiqRbe/o8jPXTe+EK4xDjECHOxdYRYc8++UhyfgXHma5w/Z5mJ+H63T3ChN3Y6O/guMcxj8NGicLDgYyQ3CKcnsUbMBuoa7j48ZgD+erqdczqbsYTpulj3LSu2POBfCQ58pn0EH1OwoTafwvX1+JV2VmIxEwHlJlBsdkwLHy2mZjcgjI9kJ4Ynbh6/Xu4l09YfhPjCsSJg7hpIbbng/92M5Mjn0kPcdlJGF/7JQJCSrsgAseeHzoqL+4bFnSe5EJKzgHpeaTsg3v9rCrtYFz+hScZdzAGYs8HX84H9Jn0KAYnQfyuIQT4Y5mo0akiMhQeDh44tEguXGcE0iP845MvxxzEjRs3QZ5Ux3hCtnUxbqq6PR/8cRdAcuSz1YfzGEhNm2BdDfjkvw0LcTYKokCK+oaFAolIjiDFBYl02/oujDmQC1c+ZxzC+BoIp2t35HXHPrDnA/lIcuQz6SKOOAnWVqsRbHscjidDNf0gRWF7CNX2M1l3VTOQbmpd55gDqT01xDhkmBTiJMhGsB+isdrPbGe6wrU15RjIzkQEyHB3GqYbYCAiSeHwCMBmI7mAYiwt6grX7QT9h5dHHcQ/P/sKlEm7GYd37lHGGaLut2tbirD5iT6TriCuKsVJsLrCwyWuih2Yj/unMC2VFlfsgr5hodxsZHIEZVoTkP787APw7TXHZy/ac/25rJ3pSpP24tRrZnyeW012bbtZbS9AefKZ+b6mMtjJS6V6GP/zOR3wK+pkQn7bzHbJCCRDsqFlBpz+djHCV7a2wMUr/x0xiM++ugprq45bnFhbhdNoF+MKLOt32C75SvqIb7xUO3/Fdr/8uMqDLmsqwU3VipH2QzA2k3hTr11ICnqZHMn7F+HCFIfZQQ5JfDVUvW1mzv708/V316FV/wF4Je9hsgSv3GOMYz71Jg6bkezS0CN5N1WLhSOussW2jResrnzNZXUFm5PnW0nl2CciVLQHebHBJh9U0g1S3GYQD4eQjH2QWH0C0utw15DXAEIybD0nxoUsYPMZmz4N59HYE+K0SzyC2Mo3bIHw4zTT+Kt33ESAX/FZCMWovUtMIMzvHRFKJA9G+VAGvJ7IPsKGC3HdDYI4qnwzhJQZmQ5l2AODcMSWb6mJ6fgWn+H4bsxbWzX9tmt2l9Xl7fzYcpwJGhl5MI5XESoL8kaGKB9XWww8xOoYIXBrD3hvOgnK9BbEYdypHsctSBcGYLbJ+FMvbupz2AanJ01uAPLVJab88B03H1xidKH8WB0TCCq1KNEM4YgRDm7FRlys+m8L6G6gJLmPkpuqxhJU0st8JF8FMeV+dwTipFL9zDlGewmB1wYdzJh/qRlccntHDcqevBCv6NBZ3xIz+CGP5xYTKIoMIMZzo+UTIAK3WRKgULUB+egcrTs/7A06XpQ20Tlai+O4mm0DKLuSAgPwkWgqIcOkkC+BOBRdVlcC+ciL0kUNG4jodd3vnKM13yHAK/8UBG6nTBrBOUc/pfDBRZJ88cg9DuQbL1rzxdw3yx61exPbOUazi4Rd8VqYMhBIwyunF5yz9VMCUV6vxQ+ECJcH8s05SlMy4t145xi1jAkjfIu7GIESxzYPSacC1Gfkg3fhGbD6ddMlVvuCQz/0oHAfKclSmiAAK0JN75zdC/Oy9JMKanKyTxBvOGAJJEbd4fAvVrxo9UukxMfZwbu4hwWiKDLCXCSfTNAUTba9Cs5x1SD4OBwIm4qjNQOkKE1uBH+aQkssVZmbqZ8UCLAvyS5BnLDf2hvaE6P+MZQfpYngsuBd2A1+W7EqBUZ4MUM/KXAvMjGbHvm23gCXaI1yTD9Po7KezWBJB8EXp0ACD0s+J6NnQkGzJGdPlFDHBdI+5t/Z+dGaQC4bHpvOgg+uznJcIGereiYUykIjs+WW22mrBi9WLbqnJx9wlugkIlHifvBGcgLNKLPQ4ESA+pCzI4jfwy2Ajff8CAduWzy4rLjnnWEGqFdmpfdMCKgaZEOZc5qrxg3nWM28cXmohhetPcqqsn4veG02MczDmWVmWs+4wjmr18YvWFfLBVI3bk8HubxZ5spVRZHTyQzJsSovoPHxhAKrQdyKrFNcED/wo8pnjuvzWrgHayJyIY5bz2ITw1ycJp9P7R4X8LDCHK/L2l0sEH60tmrcHzzjRet4tM9hVck+xQzKNxnGLRDqO+KUZZ7gqnHdZY1mxoQ8QUfjlYwI1taCBy5YBKrKcynd9wTqNwufEfhrqq17Ko16wh4FpPFK45ZtKDNOgnshZjDfAH9M7r4nyPONjEua/hZXjav8NzTTJvThTF6UppJtF+JqwA2NE15U6eFZdGgsmJvRyziUeBXIX7PT2huazRP+lKkgavszeM18jW0oVcfBrYCqYoRnN3aPGlw1iMM17ai1Gtqvnd/Q/H5SnvvF7f12ljkcz0psUmWBpSoz0LnRgKpBugq6L8CuxSkQde6kPcAsWqN7Ao1+yzaUacdAsckI0jwDPJPU5TBmbOxi/UW64pQOrjc+5/1V/dtJfRIbrw0KWFVWV+Hw6GNDZE6aHp7e0OUQ5qTrmY48rw/4sRWW3ojSpk36I+Wzo7Y/7hyl+ZJtXVI7WJ+45hrgacz29A32QTISrCDpiJLbuWp8Oiuh8jGYiof8eTHqDEtVKkCGmZVZqzI9scsuSIZkZXTfKnYHt8NNmLK3FaQxpb9GJz5jVcHMclWhrD+VeHfQsJLkWqohTGrlqnFZ9LrukSl97YIXpU5kVcHMSvDKTppnhNmY8WkJXXcFnSMZSY6e3cO1ruKxU/7+CGUSnbnCti4bWjHbOAvlGOApdPrJ9beDjtE5khFsaOaq8dHzMaW/vC/e6KGMWm4flYMku4cNnVmpPej8udtA1aBzrll47RGjs/aG+vX75tUkyihl1lKVZnDFrIuy+2AaOv9EvAX0nY7ROZeEJq4aF+g3zPvqHStejOYvlvGuA1FmNxtCM1P18AcMgjALv9MxYWaX9WcBktWuuu9eFqPM4mbvAzbEEg5h9tHpLIOtP+g7HeMnNHLVeG/JkvF7YWxc33jDqqy0ZhoEKovzM1P0DPSdjtFvG5ZVXLP0vn19z3KrVTvIHF3fYHHeCvruHN/AbdNN3PO69+17iLgzjrRux8El/SwIMg0M9P3HG9HqsPv+hUrrJXEvczj+AAbRx+AcX88F0v1AvBnKAnlTG8Rln5/6LuLHW5/zorT+D0wg1qq8y5xfu88CSyCnH5h3dW/ZGXve8uOMZRWP0no8cIFY7+YfswURrT36QL09ffsMppHYegW/P7CBWHvlMOGBe5/9jtdjY7R8wkTb+R9meZA6n2oJWAAAAABJRU5ErkJggg==";
  } else {
    L.Icon.Default.prototype.options.iconUrl = "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABkAAAApCAYAAADAk4LOAAAGmklEQVRYw7VXeUyTZxjvNnfELFuyIzOabermMZEeQC/OclkO49CpOHXOLJl/CAURuYbQi3KLgEhbrhZ1aDwmaoGqKII6odATmH/scDFbdC7LvFqOCc+e95s2VG50X/LLm/f4/Z7neY/ne18aANCmAr5E/xZf1uDOkTcGcWR6hl9247tT5U7Y6SNvWsKT63P58qbfeLJG8M5qcgTknrvvrdDbsT7Ml+tv82X6vVxJE33aRmgSyYtcWVMqX97Yv2JvW39UhRE2HuyBL+t+gK1116ly06EeWFNlAmHxlQE0OMiV6mQCScusKRlhS3QLeVJdl1+23h5dY4FNB3thrbYboqptEFlphTC1hSpJnbRvxP4NWgsE5Jyz86QNNi/5qSUTGuFk1gu54tN9wuK2wc3o+Wc13RCmsoBwEqzGcZsxsvCSy/9wJKf7UWf1mEY8JWfewc67UUoDbDjQC+FqK4QqLVMGGR9d2wurKzqBk3nqIT/9zLxRRjgZ9bqQgub+DdoeCC03Q8j+0QhFhBHR/eP3U/zCln7Uu+hihJ1+bBNffLIvmkyP0gpBZWYXhKussK6mBz5HT6M1Nqpcp+mBCPXosYQfrekGvrjewd59/GvKCE7TbK/04/ZV5QZYVWmDwH1mF3xa2Q3ra3DBC5vBT1oP7PTj4C0+CcL8c7C2CtejqhuCnuIQHaKHzvcRfZpnylFfXsYJx3pNLwhKzRAwAhEqG0SpusBHfAKkxw3w4627MPhoCH798z7s0ZnBJ/MEJbZSbXPhER2ih7p2ok/zSj2cEJDd4CAe+5WYnBCgR2uruyEw6zRoW6/DWJ/OeAP8pd/BGtzOZKpG8oke0SX6GMmRk6GFlyAc59K32OTEinILRJRchah8HQwND8N435Z9Z0FY1EqtxUg+0SO6RJ/mmXz4VuS+DpxXC3gXmZwIL7dBSH4zKE50wESf8qwVgrP1EIlTO5JP9Igu0aexdh28F1lmAEGJGfh7jE6ElyM5Rw/FDcYJjWhbeiBYoYNIpc2FT/SILivp0F1ipDWk4BIEo2VuodEJUifhbiltnNBIXPUFCMpthtAyqws/BPlEF/VbaIxErdxPphsU7rcCp8DohC+GvBIPJS/tW2jtvTmmAeuNO8BNOYQeG8G/2OzCJ3q+soYB5i6NhMaKr17FSal7GIHheuV3uSCY8qYVuEm1cOzqdWr7ku/R0BDoTT+DT+ohCM6/CCvKLKO4RI+dXPeAuaMqksaKrZ7L3FE5FIFbkIceeOZ2OcHO6wIhTkNo0ffgjRGxEqogXHYUPHfWAC/lADpwGcLRY3aeK4/oRGCKYcZXPVoeX/kelVYY8dUGf8V5EBRbgJXT5QIPhP9ePJi428JKOiEYhYXFBqou2Guh+p/mEB1/RfMw6rY7cxcjTrneI1FrDyuzUSRm9miwEJx8E/gUmqlyvHGkneiwErR21F3tNOK5Tf0yXaT+O7DgCvALTUBXdM4YhC/IawPU+2PduqMvuaR6eoxSwUk75ggqsYJ7VicsnwGIkZBSXKOUww73WGXyqP+J2/b9c+gi1YAg/xpwck3gJuucNrh5JvDPvQr0WFXf0piyt8f8/WI0hV4pRxxkQZdJDfDJNOAmM0Ag8jyT6hz0WGXWuP94Yh2jcfjmXAGvHCMslRimDHYuHuDsy2QtHuIavznhbYURq5R57KpzBBRZKPJi8eQg48h4j8SDdowifdIrEVdU+gbO6QNvRRt4ZBthUaZhUnjlYObNagV3keoeru3rU7rcuceqU1mJBxy+BWZYlNEBH+0eH4vRiB+OYybU2hnblYlTvkHinM4m54YnxSyaZYSF6R3jwgP7udKLGIX6r/lbNa9N6y5MFynjWDtrHd75ZvTYAPO/6RgF0k76mQla3FGq7dO+cH8sKn0Vo7nDllwAhqwLPkxrHwWmHJOo+AKJ4rab5OgrM7rVu8eWb2Pu0Dh4eDgXoOfvp7Y7QeqknRmvcTBEyq9m/HQQSCSz6LHq3z0yzsNySRfMS253wl2KyRDbcZPcfJKjZmSEOjcxyi+Y8dUOtsIEH6R2wNykdqrkYJ0RV92H0W58pkfQk7cKevsLK10Py8SdMGfXNXATY+pPbyJR/ET6n9nIfztNtZYRV9XniQu9IA2vOVgy4ir7GCLVmmd+zjkH0eAF9Po6K61pmCXHxU5rHMYd1ftc3owjwRSVRzLjKvqZEty6cRUD7jGqiOdu5HG6MdHjNcNYGqfDm5YRzLBBCCDl/2bk8a8gdbqcfwECu62Fg/HrggAAAABJRU5ErkJggg==";
  }
  */

}


},{"./global/leaflet":10}],6:[function(require,module,exports){
"use strict";

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

// add texxtsize, textOnly, and style
_leaflet2["default"].Tooltip.prototype.options.textsize = "10px";
_leaflet2["default"].Tooltip.prototype.options.textOnly = false;
_leaflet2["default"].Tooltip.prototype.options.style = null; // copy original layout to not completely stomp it.

var initLayoutOriginal = _leaflet2["default"].Tooltip.prototype._initLayout;

_leaflet2["default"].Tooltip.prototype._initLayout = function () {
  initLayoutOriginal.call(this);
  this._container.style.fontSize = this.options.textsize;

  if (this.options.textOnly) {
    _leaflet2["default"].DomUtil.addClass(this._container, "leaflet-tooltip-text-only");
  }

  if (this.options.style) {
    for (var property in this.options.style) {
      this._container.style[property] = this.options.style[property];
    }
  }
};


},{"./global/leaflet":10}],7:[function(require,module,exports){
"use strict";

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

var protocolRegex = /^\/\//;

var upgrade_protocol = function upgrade_protocol(urlTemplate) {
  if (protocolRegex.test(urlTemplate)) {
    if (window.location.protocol === "file:") {
      // if in a local file, support http
      // http should auto upgrade if necessary
      urlTemplate = "http:" + urlTemplate;
    }
  }

  return urlTemplate;
};

var originalLTileLayerInitialize = _leaflet2["default"].TileLayer.prototype.initialize;

_leaflet2["default"].TileLayer.prototype.initialize = function (urlTemplate, options) {
  urlTemplate = upgrade_protocol(urlTemplate);
  originalLTileLayerInitialize.call(this, urlTemplate, options);
};

var originalLTileLayerWMSInitialize = _leaflet2["default"].TileLayer.WMS.prototype.initialize;

_leaflet2["default"].TileLayer.WMS.prototype.initialize = function (urlTemplate, options) {
  urlTemplate = upgrade_protocol(urlTemplate);
  originalLTileLayerWMSInitialize.call(this, urlTemplate, options);
};


},{"./global/leaflet":10}],8:[function(require,module,exports){
(function (global){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = global.HTMLWidgets;


}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{}],9:[function(require,module,exports){
(function (global){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = global.jQuery;


}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{}],10:[function(require,module,exports){
(function (global){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = global.L;


}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{}],11:[function(require,module,exports){
(function (global){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = global.L.Proj;


}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{}],12:[function(require,module,exports){
(function (global){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = global.Shiny;


}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{}],13:[function(require,module,exports){
"use strict";

var _jquery = require("./global/jquery");

var _jquery2 = _interopRequireDefault(_jquery);

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

var _shiny = require("./global/shiny");

var _shiny2 = _interopRequireDefault(_shiny);

var _htmlwidgets = require("./global/htmlwidgets");

var _htmlwidgets2 = _interopRequireDefault(_htmlwidgets);

var _util = require("./util");

var _crs_utils = require("./crs_utils");

var _controlStore = require("./control-store");

var _controlStore2 = _interopRequireDefault(_controlStore);

var _layerManager = require("./layer-manager");

var _layerManager2 = _interopRequireDefault(_layerManager);

var _methods = require("./methods");

var _methods2 = _interopRequireDefault(_methods);

require("./fixup-default-icon");

require("./fixup-default-tooltip");

require("./fixup-url-protocol");

var _dataframe = require("./dataframe");

var _dataframe2 = _interopRequireDefault(_dataframe);

var _clusterLayerStore = require("./cluster-layer-store");

var _clusterLayerStore2 = _interopRequireDefault(_clusterLayerStore);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

window.LeafletWidget = {};
window.LeafletWidget.utils = {};

var methods = window.LeafletWidget.methods = _jquery2["default"].extend({}, _methods2["default"]);

window.LeafletWidget.DataFrame = _dataframe2["default"];
window.LeafletWidget.ClusterLayerStore = _clusterLayerStore2["default"];
window.LeafletWidget.utils.getCRS = _crs_utils.getCRS; // Send updated bounds back to app. Takes a leaflet event object as input.

function updateBounds(map) {
  var id = map.getContainer().id;
  var bounds = map.getBounds();

  _shiny2["default"].onInputChange(id + "_bounds", {
    north: bounds.getNorthEast().lat,
    east: bounds.getNorthEast().lng,
    south: bounds.getSouthWest().lat,
    west: bounds.getSouthWest().lng
  });

  _shiny2["default"].onInputChange(id + "_center", {
    lng: map.getCenter().lng,
    lat: map.getCenter().lat
  });

  _shiny2["default"].onInputChange(id + "_zoom", map.getZoom());
}

function preventUnintendedZoomOnScroll(map) {
  // Prevent unwanted scroll capturing. Similar in purpose to
  // https://github.com/CliffCloud/Leaflet.Sleep but with a
  // different set of heuristics.
  // The basic idea is that when a mousewheel/DOMMouseScroll
  // event is seen, we disable scroll wheel zooming until the
  // user moves their mouse cursor or clicks on the map. This
  // is slightly trickier than just listening for mousemove,
  // because mousemove is fired when the page is scrolled,
  // even if the user did not physically move the mouse. We
  // handle this by examining the mousemove event's screenX
  // and screenY properties; if they change, we know it's a
  // "true" move.
  // lastScreen can never be null, but its x and y can.
  var lastScreen = {
    x: null,
    y: null
  };
  (0, _jquery2["default"])(document).on("mousewheel DOMMouseScroll", "*", function (e) {
    // Disable zooming (until the mouse moves or click)
    map.scrollWheelZoom.disable(); // Any mousemove events at this screen position will be ignored.

    lastScreen = {
      x: e.originalEvent.screenX,
      y: e.originalEvent.screenY
    };
  });
  (0, _jquery2["default"])(document).on("mousemove", "*", function (e) {
    // Did the mouse really move?
    if (lastScreen.x !== null && e.screenX !== lastScreen.x || e.screenY !== lastScreen.y) {
      // It really moved. Enable zooming.
      map.scrollWheelZoom.enable();
      lastScreen = {
        x: null,
        y: null
      };
    }
  });
  (0, _jquery2["default"])(document).on("mousedown", ".leaflet", function (e) {
    // Clicking always enables zooming.
    map.scrollWheelZoom.enable();
    lastScreen = {
      x: null,
      y: null
    };
  });
}

_htmlwidgets2["default"].widget({
  name: "leaflet",
  type: "output",
  factory: function factory(el, width, height) {
    var map = null;
    return {
      // we need to store our map in our returned object.
      getMap: function getMap() {
        return map;
      },
      renderValue: function renderValue(data) {
        // Create an appropriate CRS Object if specified
        if (data && data.options && data.options.crs) {
          data.options.crs = (0, _crs_utils.getCRS)(data.options.crs);
        } // As per https://github.com/rstudio/leaflet/pull/294#discussion_r79584810


        if (map) {
          map.remove();

          map = function () {
            return;
          }(); // undefine map

        }

        if (data.options.mapFactory && typeof data.options.mapFactory === "function") {
          map = data.options.mapFactory(el, data.options);
        } else {
          map = _leaflet2["default"].map(el, data.options);
        }

        preventUnintendedZoomOnScroll(map); // Store some state in the map object

        map.leafletr = {
          // Has the map ever rendered successfully?
          hasRendered: false,
          // Data to be rendered when resize is called with area != 0
          pendingRenderData: null
        }; // Check if the map is rendered statically (no output binding)

        if (_htmlwidgets2["default"].shinyMode && /\bshiny-bound-output\b/.test(el.className)) {
          map.id = el.id; // Store the map on the element so we can find it later by ID

          (0, _jquery2["default"])(el).data("leaflet-map", map); // When the map is clicked, send the coordinates back to the app

          map.on("click", function (e) {
            _shiny2["default"].onInputChange(map.id + "_click", {
              lat: e.latlng.lat,
              lng: e.latlng.lng,
              ".nonce": Math.random() // Force reactivity if lat/lng hasn't changed

            });
          });
          var groupTimerId = null;
          map.on("moveend", function (e) {
            updateBounds(e.target);
          }).on("layeradd layerremove", function (e) {
            // If the layer that's coming or going is a group we created, tell
            // the server.
            if (map.layerManager.getGroupNameFromLayerGroup(e.layer)) {
              // But to avoid chattiness, coalesce events
              if (groupTimerId) {
                clearTimeout(groupTimerId);
                groupTimerId = null;
              }

              groupTimerId = setTimeout(function () {
                groupTimerId = null;

                _shiny2["default"].onInputChange(map.id + "_groups", map.layerManager.getVisibleGroups());
              }, 100);
            }
          });
        }

        this.doRenderValue(data, map);
      },
      doRenderValue: function doRenderValue(data, map) {
        // Leaflet does not behave well when you set up a bunch of layers when
        // the map is not visible (width/height == 0). Popups get misaligned
        // relative to their owning markers, and the fitBounds calculations
        // are off. Therefore we wait until the map is actually showing to
        // render the value (we rely on the resize() callback being invoked
        // at the appropriate time).
        //
        // There may be an issue with leafletProxy() calls being made while
        // the map is not being viewed--not sure what the right solution is
        // there.
        if (el.offsetWidth === 0 || el.offsetHeight === 0) {
          map.leafletr.pendingRenderData = data;
          return;
        }

        map.leafletr.pendingRenderData = null; // Merge data options into defaults

        var options = _jquery2["default"].extend({
          zoomToLimits: "always"
        }, data.options);

        if (!map.layerManager) {
          map.controls = new _controlStore2["default"](map);
          map.layerManager = new _layerManager2["default"](map);
        } else {
          map.controls.clear();
          map.layerManager.clear();
        }

        var explicitView = false;

        if (data.setView) {
          explicitView = true;
          map.setView.apply(map, data.setView);
        }

        if (data.fitBounds) {
          explicitView = true;
          methods.fitBounds.apply(map, data.fitBounds);
        }

        if (data.flyTo) {
          if (!explicitView && !map.leafletr.hasRendered) {
            // must be done to give a initial starting point
            map.fitWorld();
          }

          explicitView = true;
          map.flyTo.apply(map, data.flyTo);
        }

        if (data.flyToBounds) {
          if (!explicitView && !map.leafletr.hasRendered) {
            // must be done to give a initial starting point
            map.fitWorld();
          }

          explicitView = true;
          methods.flyToBounds.apply(map, data.flyToBounds);
        }

        if (data.options.center) {
          explicitView = true;
        } // Returns true if the zoomToLimits option says that the map should be
        // zoomed to map elements.


        function needsZoom() {
          return options.zoomToLimits === "always" || options.zoomToLimits === "first" && !map.leafletr.hasRendered;
        }

        if (!explicitView && needsZoom() && !map.getZoom()) {
          if (data.limits && !_jquery2["default"].isEmptyObject(data.limits)) {
            // Use the natural limits of what's being drawn on the map
            // If the size of the bounding box is 0, leaflet gets all weird
            var pad = 0.006;

            if (data.limits.lat[0] === data.limits.lat[1]) {
              data.limits.lat[0] = data.limits.lat[0] - pad;
              data.limits.lat[1] = data.limits.lat[1] + pad;
            }

            if (data.limits.lng[0] === data.limits.lng[1]) {
              data.limits.lng[0] = data.limits.lng[0] - pad;
              data.limits.lng[1] = data.limits.lng[1] + pad;
            }

            map.fitBounds([[data.limits.lat[0], data.limits.lng[0]], [data.limits.lat[1], data.limits.lng[1]]]);
          } else {
            map.fitWorld();
          }
        }

        for (var i = 0; data.calls && i < data.calls.length; i++) {
          var call = data.calls[i];
          if (methods[call.method]) methods[call.method].apply(map, call.args);else (0, _util.log)("Unknown method " + call.method);
        }

        map.leafletr.hasRendered = true;

        if (_htmlwidgets2["default"].shinyMode) {
          setTimeout(function () {
            updateBounds(map);
          }, 1);
        }
      },
      resize: function resize(width, height) {
        if (map) {
          map.invalidateSize();

          if (map.leafletr.pendingRenderData) {
            this.doRenderValue(map.leafletr.pendingRenderData, map);
          }
        }
      }
    };
  }
});

if (_htmlwidgets2["default"].shinyMode) {
  _shiny2["default"].addCustomMessageHandler("leaflet-calls", function (data) {
    var id = data.id;
    var el = document.getElementById(id);
    var map = el ? (0, _jquery2["default"])(el).data("leaflet-map") : null;

    if (!map) {
      (0, _util.log)("Couldn't find map with id " + id);
      return;
    }

    for (var i = 0; i < data.calls.length; i++) {
      var call = data.calls[i];

      if (call.dependencies) {
        _shiny2["default"].renderDependencies(call.dependencies);
      }

      if (methods[call.method]) methods[call.method].apply(map, call.args);else (0, _util.log)("Unknown method " + call.method);
    }
  });
}


},{"./cluster-layer-store":1,"./control-store":2,"./crs_utils":3,"./dataframe":4,"./fixup-default-icon":5,"./fixup-default-tooltip":6,"./fixup-url-protocol":7,"./global/htmlwidgets":8,"./global/jquery":9,"./global/leaflet":10,"./global/shiny":12,"./layer-manager":14,"./methods":15,"./util":17}],14:[function(require,module,exports){
(function (global){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = undefined;

var _jquery = require("./global/jquery");

var _jquery2 = _interopRequireDefault(_jquery);

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

var _util = require("./util");

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

var LayerManager = /*#__PURE__*/function () {
  function LayerManager(map) {
    _classCallCheck(this, LayerManager);

    this._map = map; // BEGIN layer indices
    // {<groupname>: {<stamp>: layer}}

    this._byGroup = {}; // {<categoryName>: {<stamp>: layer}}

    this._byCategory = {}; // {<categoryName_layerId>: layer}

    this._byLayerId = {}; // {<stamp>: {
    //             "group": <groupname>,
    //             "layerId": <layerId>,
    //             "category": <category>,
    //             "container": <container>
    //           }
    // }

    this._byStamp = {}; // {<crosstalkGroupName>: {<key>: [<stamp>, <stamp>, ...], ...}}

    this._byCrosstalkGroup = {}; // END layer indices
    // {<categoryName>: L.layerGroup}

    this._categoryContainers = {}; // {<groupName>: L.layerGroup}

    this._groupContainers = {};
  }

  _createClass(LayerManager, [{
    key: "addLayer",
    value: function addLayer(layer, category, layerId, group, ctGroup, ctKey) {
      var _this = this;

      // Was a group provided?
      var hasId = typeof layerId === "string";
      var grouped = typeof group === "string";
      var stamp = _leaflet2["default"].Util.stamp(layer) + ""; // This will be the default layer group to add the layer to.
      // We may overwrite this let before using it (i.e. if a group is assigned).
      // This one liner creates the _categoryContainers[category] entry if it
      // doesn't already exist.

      var container = this._categoryContainers[category] = this._categoryContainers[category] || _leaflet2["default"].layerGroup().addTo(this._map);

      var oldLayer = null;

      if (hasId) {
        // First, remove any layer with the same category and layerId
        var prefixedLayerId = this._layerIdKey(category, layerId);

        oldLayer = this._byLayerId[prefixedLayerId];

        if (oldLayer) {
          this._removeLayer(oldLayer);
        } // Update layerId index


        this._byLayerId[prefixedLayerId] = layer;
      } // Update group index


      if (grouped) {
        this._byGroup[group] = this._byGroup[group] || {};
        this._byGroup[group][stamp] = layer; // Since a group is assigned, don't add the layer to the category's layer
        // group; instead, use the group's layer group.
        // This one liner creates the _groupContainers[group] entry if it doesn't
        // already exist.

        container = this.getLayerGroup(group, true);
      } // Update category index


      this._byCategory[category] = this._byCategory[category] || {};
      this._byCategory[category][stamp] = layer; // Update stamp index

      var layerInfo = this._byStamp[stamp] = {
        layer: layer,
        group: group,
        ctGroup: ctGroup,
        ctKey: ctKey,
        layerId: layerId,
        category: category,
        container: container,
        hidden: false
      }; // Update crosstalk group index

      if (ctGroup) {
        if (layer.setStyle) {
          // Need to save this info so we know what to set opacity to later
          layer.options.origOpacity = typeof layer.options.opacity !== "undefined" ? layer.options.opacity : 0.5;
          layer.options.origFillOpacity = typeof layer.options.fillOpacity !== "undefined" ? layer.options.fillOpacity : 0.2;
        }

        var ctg = this._byCrosstalkGroup[ctGroup];

        if (!ctg) {
          ctg = this._byCrosstalkGroup[ctGroup] = {};
          var crosstalk = global.crosstalk;

          var handleFilter = function handleFilter(e) {
            if (!e.value) {
              var groupKeys = Object.keys(ctg);

              for (var i = 0; i < groupKeys.length; i++) {
                var key = groupKeys[i];
                var _layerInfo = _this._byStamp[ctg[key]];

                _this._setVisibility(_layerInfo, true);
              }
            } else {
              var selectedKeys = {};

              for (var _i = 0; _i < e.value.length; _i++) {
                selectedKeys[e.value[_i]] = true;
              }

              var _groupKeys = Object.keys(ctg);

              for (var _i2 = 0; _i2 < _groupKeys.length; _i2++) {
                var _key = _groupKeys[_i2];
                var _layerInfo2 = _this._byStamp[ctg[_key]];

                _this._setVisibility(_layerInfo2, selectedKeys[_groupKeys[_i2]]);
              }
            }
          };

          var filterHandle = new crosstalk.FilterHandle(ctGroup);
          filterHandle.on("change", handleFilter);

          var handleSelection = function handleSelection(e) {
            if (!e.value || !e.value.length) {
              var groupKeys = Object.keys(ctg);

              for (var i = 0; i < groupKeys.length; i++) {
                var key = groupKeys[i];
                var _layerInfo3 = _this._byStamp[ctg[key]];

                _this._setOpacity(_layerInfo3, 1.0);
              }
            } else {
              var selectedKeys = {};

              for (var _i3 = 0; _i3 < e.value.length; _i3++) {
                selectedKeys[e.value[_i3]] = true;
              }

              var _groupKeys2 = Object.keys(ctg);

              for (var _i4 = 0; _i4 < _groupKeys2.length; _i4++) {
                var _key2 = _groupKeys2[_i4];
                var _layerInfo4 = _this._byStamp[ctg[_key2]];

                _this._setOpacity(_layerInfo4, selectedKeys[_groupKeys2[_i4]] ? 1.0 : 0.2);
              }
            }
          };

          var selHandle = new crosstalk.SelectionHandle(ctGroup);
          selHandle.on("change", handleSelection);
          setTimeout(function () {
            handleFilter({
              value: filterHandle.filteredKeys
            });
            handleSelection({
              value: selHandle.value
            });
          }, 100);
        }

        if (!ctg[ctKey]) ctg[ctKey] = [];
        ctg[ctKey].push(stamp);
      } // Add to container


      if (!layerInfo.hidden) container.addLayer(layer);
      return oldLayer;
    }
  }, {
    key: "brush",
    value: function brush(bounds, extraInfo) {
      var _this2 = this;

      /* eslint-disable no-console */
      // For each Crosstalk group...
      Object.keys(this._byCrosstalkGroup).forEach(function (ctGroupName) {
        var ctg = _this2._byCrosstalkGroup[ctGroupName];
        var selection = []; // ...iterate over each Crosstalk key (each of which may have multiple
        // layers)...

        Object.keys(ctg).forEach(function (ctKey) {
          // ...and for each layer...
          ctg[ctKey].forEach(function (stamp) {
            var layerInfo = _this2._byStamp[stamp]; // ...if it's something with a point...

            if (layerInfo.layer.getLatLng) {
              // ... and it's inside the selection bounds...
              // TODO: Use pixel containment, not lat/lng containment
              if (bounds.contains(layerInfo.layer.getLatLng())) {
                // ...add the key to the selection.
                selection.push(ctKey);
              }
            }
          });
        });
        new global.crosstalk.SelectionHandle(ctGroupName).set(selection, extraInfo);
      });
    }
  }, {
    key: "unbrush",
    value: function unbrush(extraInfo) {
      Object.keys(this._byCrosstalkGroup).forEach(function (ctGroupName) {
        new global.crosstalk.SelectionHandle(ctGroupName).clear(extraInfo);
      });
    }
  }, {
    key: "_setVisibility",
    value: function _setVisibility(layerInfo, visible) {
      if (layerInfo.hidden ^ visible) {
        return;
      } else if (visible) {
        layerInfo.container.addLayer(layerInfo.layer);
        layerInfo.hidden = false;
      } else {
        layerInfo.container.removeLayer(layerInfo.layer);
        layerInfo.hidden = true;
      }
    }
  }, {
    key: "_setOpacity",
    value: function _setOpacity(layerInfo, opacity) {
      if (layerInfo.layer.setOpacity) {
        layerInfo.layer.setOpacity(opacity);
      } else if (layerInfo.layer.setStyle) {
        layerInfo.layer.setStyle({
          opacity: opacity * layerInfo.layer.options.origOpacity,
          fillOpacity: opacity * layerInfo.layer.options.origFillOpacity
        });
      }
    }
  }, {
    key: "getLayer",
    value: function getLayer(category, layerId) {
      return this._byLayerId[this._layerIdKey(category, layerId)];
    }
  }, {
    key: "removeLayer",
    value: function removeLayer(category, layerIds) {
      var _this3 = this;

      // Find layer info
      _jquery2["default"].each((0, _util.asArray)(layerIds), function (i, layerId) {
        var layer = _this3._byLayerId[_this3._layerIdKey(category, layerId)];

        if (layer) {
          _this3._removeLayer(layer);
        }
      });
    }
  }, {
    key: "clearLayers",
    value: function clearLayers(category) {
      var _this4 = this;

      // Find all layers in _byCategory[category]
      var catTable = this._byCategory[category];

      if (!catTable) {
        return false;
      } // Remove all layers. Make copy of keys to avoid mutating the collection
      // behind the iterator you're accessing.


      var stamps = [];

      _jquery2["default"].each(catTable, function (k, v) {
        stamps.push(k);
      });

      _jquery2["default"].each(stamps, function (i, stamp) {
        _this4._removeLayer(stamp);
      });
    }
  }, {
    key: "getLayerGroup",
    value: function getLayerGroup(group, ensureExists) {
      var g = this._groupContainers[group];

      if (ensureExists && !g) {
        this._byGroup[group] = this._byGroup[group] || {};
        g = this._groupContainers[group] = _leaflet2["default"].featureGroup();
        g.groupname = group;
        g.addTo(this._map);
      }

      return g;
    }
  }, {
    key: "getGroupNameFromLayerGroup",
    value: function getGroupNameFromLayerGroup(layerGroup) {
      return layerGroup.groupname;
    }
  }, {
    key: "getVisibleGroups",
    value: function getVisibleGroups() {
      var _this5 = this;

      var result = [];

      _jquery2["default"].each(this._groupContainers, function (k, v) {
        if (_this5._map.hasLayer(v)) {
          result.push(k);
        }
      });

      return result;
    }
  }, {
    key: "getAllGroupNames",
    value: function getAllGroupNames() {
      var result = [];

      _jquery2["default"].each(this._groupContainers, function (k, v) {
        result.push(k);
      });

      return result;
    }
  }, {
    key: "clearGroup",
    value: function clearGroup(group) {
      var _this6 = this;

      // Find all layers in _byGroup[group]
      var groupTable = this._byGroup[group];

      if (!groupTable) {
        return false;
      } // Remove all layers. Make copy of keys to avoid mutating the collection
      // behind the iterator you're accessing.


      var stamps = [];

      _jquery2["default"].each(groupTable, function (k, v) {
        stamps.push(k);
      });

      _jquery2["default"].each(stamps, function (i, stamp) {
        _this6._removeLayer(stamp);
      });
    }
  }, {
    key: "clear",
    value: function clear() {
      function clearLayerGroup(key, layerGroup) {
        layerGroup.clearLayers();
      } // Clear all indices and layerGroups


      this._byGroup = {};
      this._byCategory = {};
      this._byLayerId = {};
      this._byStamp = {};
      this._byCrosstalkGroup = {};

      _jquery2["default"].each(this._categoryContainers, clearLayerGroup);

      this._categoryContainers = {};

      _jquery2["default"].each(this._groupContainers, clearLayerGroup);

      this._groupContainers = {};
    }
  }, {
    key: "_removeLayer",
    value: function _removeLayer(layer) {
      var stamp;

      if (typeof layer === "string") {
        stamp = layer;
      } else {
        stamp = _leaflet2["default"].Util.stamp(layer);
      }

      var layerInfo = this._byStamp[stamp];

      if (!layerInfo) {
        return false;
      }

      layerInfo.container.removeLayer(stamp);

      if (typeof layerInfo.group === "string") {
        delete this._byGroup[layerInfo.group][stamp];
      }

      if (typeof layerInfo.layerId === "string") {
        delete this._byLayerId[this._layerIdKey(layerInfo.category, layerInfo.layerId)];
      }

      delete this._byCategory[layerInfo.category][stamp];
      delete this._byStamp[stamp];

      if (layerInfo.ctGroup) {
        var ctGroup = this._byCrosstalkGroup[layerInfo.ctGroup];
        var layersForKey = ctGroup[layerInfo.ctKey];
        var idx = layersForKey ? layersForKey.indexOf(stamp) : -1;

        if (idx >= 0) {
          if (layersForKey.length === 1) {
            delete ctGroup[layerInfo.ctKey];
          } else {
            layersForKey.splice(idx, 1);
          }
        }
      }
    }
  }, {
    key: "_layerIdKey",
    value: function _layerIdKey(category, layerId) {
      return category + "\n" + layerId;
    }
  }]);

  return LayerManager;
}();

exports["default"] = LayerManager;


}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{"./global/jquery":9,"./global/leaflet":10,"./util":17}],15:[function(require,module,exports){
(function (global){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});

var _jquery = require("./global/jquery");

var _jquery2 = _interopRequireDefault(_jquery);

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

var _shiny = require("./global/shiny");

var _shiny2 = _interopRequireDefault(_shiny);

var _htmlwidgets = require("./global/htmlwidgets");

var _htmlwidgets2 = _interopRequireDefault(_htmlwidgets);

var _util = require("./util");

var _crs_utils = require("./crs_utils");

var _dataframe = require("./dataframe");

var _dataframe2 = _interopRequireDefault(_dataframe);

var _clusterLayerStore = require("./cluster-layer-store");

var _clusterLayerStore2 = _interopRequireDefault(_clusterLayerStore);

var _mipmapper = require("./mipmapper");

var _mipmapper2 = _interopRequireDefault(_mipmapper);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

var methods = {};
exports["default"] = methods;

function mouseHandler(mapId, layerId, group, eventName, extraInfo) {
  return function (e) {
    if (!_htmlwidgets2["default"].shinyMode) return;
    var latLng = e.target.getLatLng ? e.target.getLatLng() : e.latlng;

    if (latLng) {
      // retrieve only lat, lon values to remove prototype
      //   and extra parameters added by 3rd party modules
      // these objects are for json serialization, not javascript
      var latLngVal = _leaflet2["default"].latLng(latLng); // make sure it has consistent shape


      latLng = {
        lat: latLngVal.lat,
        lng: latLngVal.lng
      };
    }

    var eventInfo = _jquery2["default"].extend({
      id: layerId,
      ".nonce": Math.random() // force reactivity

    }, group !== null ? {
      group: group
    } : null, latLng, extraInfo);

    _shiny2["default"].onInputChange(mapId + "_" + eventName, eventInfo);
  };
}

methods.mouseHandler = mouseHandler;

methods.clearGroup = function (group) {
  var _this = this;

  _jquery2["default"].each((0, _util.asArray)(group), function (i, v) {
    _this.layerManager.clearGroup(v);
  });
};

methods.setView = function (center, zoom, options) {
  this.setView(center, zoom, options);
};

methods.fitBounds = function (lat1, lng1, lat2, lng2, options) {
  this.fitBounds([[lat1, lng1], [lat2, lng2]], options);
};

methods.flyTo = function (center, zoom, options) {
  this.flyTo(center, zoom, options);
};

methods.flyToBounds = function (lat1, lng1, lat2, lng2, options) {
  this.flyToBounds([[lat1, lng1], [lat2, lng2]], options);
};

methods.setMaxBounds = function (lat1, lng1, lat2, lng2) {
  this.setMaxBounds([[lat1, lng1], [lat2, lng2]]);
};

methods.addPopups = function (lat, lng, popup, layerId, group, options) {
  var _this2 = this;

  var df = new _dataframe2["default"]().col("lat", lat).col("lng", lng).col("popup", popup).col("layerId", layerId).col("group", group).cbind(options);

  var _loop = function _loop(i) {
    if (_jquery2["default"].isNumeric(df.get(i, "lat")) && _jquery2["default"].isNumeric(df.get(i, "lng"))) {
      (function () {
        var popup = _leaflet2["default"].popup(df.get(i)).setLatLng([df.get(i, "lat"), df.get(i, "lng")]).setContent(df.get(i, "popup"));

        var thisId = df.get(i, "layerId");
        var thisGroup = df.get(i, "group");
        this.layerManager.addLayer(popup, "popup", thisId, thisGroup);
      }).call(_this2);
    }
  };

  for (var i = 0; i < df.nrow(); i++) {
    _loop(i);
  }
};

methods.removePopup = function (layerId) {
  this.layerManager.removeLayer("popup", layerId);
};

methods.clearPopups = function () {
  this.layerManager.clearLayers("popup");
};

methods.addTiles = function (urlTemplate, layerId, group, options) {
  this.layerManager.addLayer(_leaflet2["default"].tileLayer(urlTemplate, options), "tile", layerId, group);
};

methods.removeTiles = function (layerId) {
  this.layerManager.removeLayer("tile", layerId);
};

methods.clearTiles = function () {
  this.layerManager.clearLayers("tile");
};

methods.addWMSTiles = function (baseUrl, layerId, group, options) {
  if (options && options.crs) {
    options.crs = (0, _crs_utils.getCRS)(options.crs);
  }

  this.layerManager.addLayer(_leaflet2["default"].tileLayer.wms(baseUrl, options), "tile", layerId, group);
}; // Given:
//   {data: ["a", "b", "c"], index: [0, 1, 0, 2]}
// returns:
//   ["a", "b", "a", "c"]


function unpackStrings(iconset) {
  if (!iconset) {
    return iconset;
  }

  if (typeof iconset.index === "undefined") {
    return iconset;
  }

  iconset.data = (0, _util.asArray)(iconset.data);
  iconset.index = (0, _util.asArray)(iconset.index);
  return _jquery2["default"].map(iconset.index, function (e, i) {
    return iconset.data[e];
  });
}

function addMarkers(map, df, group, clusterOptions, clusterId, markerFunc) {
  (function () {
    var _this3 = this;

    var clusterGroup = this.layerManager.getLayer("cluster", clusterId),
        cluster = clusterOptions !== null;

    if (cluster && !clusterGroup) {
      clusterGroup = _leaflet2["default"].markerClusterGroup.layerSupport(clusterOptions);

      if (clusterOptions.freezeAtZoom) {
        var freezeAtZoom = clusterOptions.freezeAtZoom;
        delete clusterOptions.freezeAtZoom;
        clusterGroup.freezeAtZoom(freezeAtZoom);
      }

      clusterGroup.clusterLayerStore = new _clusterLayerStore2["default"](clusterGroup);
    }

    var extraInfo = cluster ? {
      clusterId: clusterId
    } : {};

    var _loop2 = function _loop2(i) {
      if (_jquery2["default"].isNumeric(df.get(i, "lat")) && _jquery2["default"].isNumeric(df.get(i, "lng"))) {
        (function () {
          var marker = markerFunc(df, i);
          var thisId = df.get(i, "layerId");
          var thisGroup = cluster ? null : df.get(i, "group");

          if (cluster) {
            clusterGroup.clusterLayerStore.add(marker, thisId);
          } else {
            this.layerManager.addLayer(marker, "marker", thisId, thisGroup, df.get(i, "ctGroup", true), df.get(i, "ctKey", true));
          }

          var popup = df.get(i, "popup");
          var popupOptions = df.get(i, "popupOptions");

          if (popup !== null) {
            if (popupOptions !== null) {
              marker.bindPopup(popup, popupOptions);
            } else {
              marker.bindPopup(popup);
            }
          }

          var label = df.get(i, "label");
          var labelOptions = df.get(i, "labelOptions");

          if (label !== null) {
            if (labelOptions !== null) {
              if (labelOptions.permanent) {
                marker.bindTooltip(label, labelOptions).openTooltip();
              } else {
                marker.bindTooltip(label, labelOptions);
              }
            } else {
              marker.bindTooltip(label);
            }
          }

          marker.on("click", mouseHandler(this.id, thisId, thisGroup, "marker_click", extraInfo), this);
          marker.on("mouseover", mouseHandler(this.id, thisId, thisGroup, "marker_mouseover", extraInfo), this);
          marker.on("mouseout", mouseHandler(this.id, thisId, thisGroup, "marker_mouseout", extraInfo), this);
          marker.on("dragend", mouseHandler(this.id, thisId, thisGroup, "marker_dragend", extraInfo), this);
        }).call(_this3);
      }
    };

    for (var i = 0; i < df.nrow(); i++) {
      _loop2(i);
    }

    if (cluster) {
      this.layerManager.addLayer(clusterGroup, "cluster", clusterId, group);
    }
  }).call(map);
}

methods.addGenericMarkers = addMarkers;

methods.addMarkers = function (lat, lng, icon, layerId, group, options, popup, popupOptions, clusterOptions, clusterId, label, labelOptions, crosstalkOptions) {
  var icondf;
  var getIcon;

  if (icon) {
    // Unpack icons
    icon.iconUrl = unpackStrings(icon.iconUrl);
    icon.iconRetinaUrl = unpackStrings(icon.iconRetinaUrl);
    icon.shadowUrl = unpackStrings(icon.shadowUrl);
    icon.shadowRetinaUrl = unpackStrings(icon.shadowRetinaUrl); // This cbinds the icon URLs and any other icon options; they're all
    // present on the icon object.

    icondf = new _dataframe2["default"]().cbind(icon); // Constructs an icon from a specified row of the icon dataframe.

    getIcon = function getIcon(i) {
      var opts = icondf.get(i);

      if (!opts.iconUrl) {
        return new _leaflet2["default"].Icon.Default();
      } // Composite options (like points or sizes) are passed from R with each
      // individual component as its own option. We need to combine them now
      // into their composite form.


      if (opts.iconWidth) {
        opts.iconSize = [opts.iconWidth, opts.iconHeight];
      }

      if (opts.shadowWidth) {
        opts.shadowSize = [opts.shadowWidth, opts.shadowHeight];
      }

      if (opts.iconAnchorX) {
        opts.iconAnchor = [opts.iconAnchorX, opts.iconAnchorY];
      }

      if (opts.shadowAnchorX) {
        opts.shadowAnchor = [opts.shadowAnchorX, opts.shadowAnchorY];
      }

      if (opts.popupAnchorX) {
        opts.popupAnchor = [opts.popupAnchorX, opts.popupAnchorY];
      }

      return new _leaflet2["default"].Icon(opts);
    };
  }

  if (!(_jquery2["default"].isEmptyObject(lat) || _jquery2["default"].isEmptyObject(lng)) || _jquery2["default"].isNumeric(lat) && _jquery2["default"].isNumeric(lng)) {
    var df = new _dataframe2["default"]().col("lat", lat).col("lng", lng).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).cbind(options).cbind(crosstalkOptions || {});
    if (icon) icondf.effectiveLength = df.nrow();
    addMarkers(this, df, group, clusterOptions, clusterId, function (df, i) {
      var options = df.get(i);
      if (icon) options.icon = getIcon(i);
      return _leaflet2["default"].marker([df.get(i, "lat"), df.get(i, "lng")], options);
    });
  }
};

methods.addAwesomeMarkers = function (lat, lng, icon, layerId, group, options, popup, popupOptions, clusterOptions, clusterId, label, labelOptions, crosstalkOptions) {
  var icondf;
  var getIcon;

  if (icon) {
    // This cbinds the icon URLs and any other icon options; they're all
    // present on the icon object.
    icondf = new _dataframe2["default"]().cbind(icon); // Constructs an icon from a specified row of the icon dataframe.

    getIcon = function getIcon(i) {
      var opts = icondf.get(i);

      if (!opts) {
        return new _leaflet2["default"].AwesomeMarkers.icon();
      }

      if (opts.squareMarker) {
        opts.className = "awesome-marker awesome-marker-square";
      }

      return new _leaflet2["default"].AwesomeMarkers.icon(opts);
    };
  }

  if (!(_jquery2["default"].isEmptyObject(lat) || _jquery2["default"].isEmptyObject(lng)) || _jquery2["default"].isNumeric(lat) && _jquery2["default"].isNumeric(lng)) {
    var df = new _dataframe2["default"]().col("lat", lat).col("lng", lng).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).cbind(options).cbind(crosstalkOptions || {});
    if (icon) icondf.effectiveLength = df.nrow();
    addMarkers(this, df, group, clusterOptions, clusterId, function (df, i) {
      var options = df.get(i);
      if (icon) options.icon = getIcon(i);
      return _leaflet2["default"].marker([df.get(i, "lat"), df.get(i, "lng")], options);
    });
  }
};

function addLayers(map, category, df, layerFunc) {
  var _loop3 = function _loop3(i) {
    (function () {
      var layer = layerFunc(df, i);

      if (!_jquery2["default"].isEmptyObject(layer)) {
        var thisId = df.get(i, "layerId");
        var thisGroup = df.get(i, "group");
        this.layerManager.addLayer(layer, category, thisId, thisGroup, df.get(i, "ctGroup", true), df.get(i, "ctKey", true));

        if (layer.bindPopup) {
          var popup = df.get(i, "popup");
          var popupOptions = df.get(i, "popupOptions");

          if (popup !== null) {
            if (popupOptions !== null) {
              layer.bindPopup(popup, popupOptions);
            } else {
              layer.bindPopup(popup);
            }
          }
        }

        if (layer.bindTooltip) {
          var label = df.get(i, "label");
          var labelOptions = df.get(i, "labelOptions");

          if (label !== null) {
            if (labelOptions !== null) {
              layer.bindTooltip(label, labelOptions);
            } else {
              layer.bindTooltip(label);
            }
          }
        }

        layer.on("click", mouseHandler(this.id, thisId, thisGroup, category + "_click"), this);
        layer.on("mouseover", mouseHandler(this.id, thisId, thisGroup, category + "_mouseover"), this);
        layer.on("mouseout", mouseHandler(this.id, thisId, thisGroup, category + "_mouseout"), this);
        var highlightStyle = df.get(i, "highlightOptions");

        if (!_jquery2["default"].isEmptyObject(highlightStyle)) {
          var defaultStyle = {};

          _jquery2["default"].each(highlightStyle, function (k, v) {
            if (k != "bringToFront" && k != "sendToBack") {
              if (df.get(i, k)) {
                defaultStyle[k] = df.get(i, k);
              }
            }
          });

          layer.on("mouseover", function (e) {
            this.setStyle(highlightStyle);

            if (highlightStyle.bringToFront) {
              this.bringToFront();
            }
          });
          layer.on("mouseout", function (e) {
            this.setStyle(defaultStyle);

            if (highlightStyle.sendToBack) {
              this.bringToBack();
            }
          });
        }
      }
    }).call(map);
  };

  for (var i = 0; i < df.nrow(); i++) {
    _loop3(i);
  }
}

methods.addGenericLayers = addLayers;

methods.addCircles = function (lat, lng, radius, layerId, group, options, popup, popupOptions, label, labelOptions, highlightOptions, crosstalkOptions) {
  if (!(_jquery2["default"].isEmptyObject(lat) || _jquery2["default"].isEmptyObject(lng)) || _jquery2["default"].isNumeric(lat) && _jquery2["default"].isNumeric(lng)) {
    var df = new _dataframe2["default"]().col("lat", lat).col("lng", lng).col("radius", radius).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).col("highlightOptions", highlightOptions).cbind(options).cbind(crosstalkOptions || {});
    addLayers(this, "shape", df, function (df, i) {
      if (_jquery2["default"].isNumeric(df.get(i, "lat")) && _jquery2["default"].isNumeric(df.get(i, "lng")) && _jquery2["default"].isNumeric(df.get(i, "radius"))) {
        return _leaflet2["default"].circle([df.get(i, "lat"), df.get(i, "lng")], df.get(i, "radius"), df.get(i));
      } else {
        return null;
      }
    });
  }
};

methods.addCircleMarkers = function (lat, lng, radius, layerId, group, options, clusterOptions, clusterId, popup, popupOptions, label, labelOptions, crosstalkOptions) {
  if (!(_jquery2["default"].isEmptyObject(lat) || _jquery2["default"].isEmptyObject(lng)) || _jquery2["default"].isNumeric(lat) && _jquery2["default"].isNumeric(lng)) {
    var df = new _dataframe2["default"]().col("lat", lat).col("lng", lng).col("radius", radius).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).cbind(crosstalkOptions || {}).cbind(options);
    addMarkers(this, df, group, clusterOptions, clusterId, function (df, i) {
      return _leaflet2["default"].circleMarker([df.get(i, "lat"), df.get(i, "lng")], df.get(i));
    });
  }
};
/*
 * @param lat Array of arrays of latitude coordinates for polylines
 * @param lng Array of arrays of longitude coordinates for polylines
 */


methods.addPolylines = function (polygons, layerId, group, options, popup, popupOptions, label, labelOptions, highlightOptions) {
  if (polygons.length > 0) {
    var df = new _dataframe2["default"]().col("shapes", polygons).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).col("highlightOptions", highlightOptions).cbind(options);
    addLayers(this, "shape", df, function (df, i) {
      var shapes = df.get(i, "shapes");
      shapes = shapes.map(function (shape) {
        return _htmlwidgets2["default"].dataframeToD3(shape[0]);
      });

      if (shapes.length > 1) {
        return _leaflet2["default"].polyline(shapes, df.get(i));
      } else {
        return _leaflet2["default"].polyline(shapes[0], df.get(i));
      }
    });
  }
};

methods.removeMarker = function (layerId) {
  this.layerManager.removeLayer("marker", layerId);
};

methods.clearMarkers = function () {
  this.layerManager.clearLayers("marker");
};

methods.removeMarkerCluster = function (layerId) {
  this.layerManager.removeLayer("cluster", layerId);
};

methods.removeMarkerFromCluster = function (layerId, clusterId) {
  var cluster = this.layerManager.getLayer("cluster", clusterId);
  if (!cluster) return;
  cluster.clusterLayerStore.remove(layerId);
};

methods.clearMarkerClusters = function () {
  this.layerManager.clearLayers("cluster");
};

methods.removeShape = function (layerId) {
  this.layerManager.removeLayer("shape", layerId);
};

methods.clearShapes = function () {
  this.layerManager.clearLayers("shape");
};

methods.addRectangles = function (lat1, lng1, lat2, lng2, layerId, group, options, popup, popupOptions, label, labelOptions, highlightOptions) {
  var df = new _dataframe2["default"]().col("lat1", lat1).col("lng1", lng1).col("lat2", lat2).col("lng2", lng2).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).col("highlightOptions", highlightOptions).cbind(options);
  addLayers(this, "shape", df, function (df, i) {
    if (_jquery2["default"].isNumeric(df.get(i, "lat1")) && _jquery2["default"].isNumeric(df.get(i, "lng1")) && _jquery2["default"].isNumeric(df.get(i, "lat2")) && _jquery2["default"].isNumeric(df.get(i, "lng2"))) {
      return _leaflet2["default"].rectangle([[df.get(i, "lat1"), df.get(i, "lng1")], [df.get(i, "lat2"), df.get(i, "lng2")]], df.get(i));
    } else {
      return null;
    }
  });
};
/*
 * @param lat Array of arrays of latitude coordinates for polygons
 * @param lng Array of arrays of longitude coordinates for polygons
 */


methods.addPolygons = function (polygons, layerId, group, options, popup, popupOptions, label, labelOptions, highlightOptions) {
  if (polygons.length > 0) {
    var df = new _dataframe2["default"]().col("shapes", polygons).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).col("highlightOptions", highlightOptions).cbind(options);
    addLayers(this, "shape", df, function (df, i) {
      // This code used to use L.multiPolygon, but that caused
      // double-click on a multipolygon to fail to zoom in on the
      // map. Surprisingly, putting all the rings in a single
      // polygon seems to still work; complicated multipolygons
      // are still rendered correctly.
      var shapes = df.get(i, "shapes").map(function (polygon) {
        return polygon.map(_htmlwidgets2["default"].dataframeToD3);
      }).reduce(function (acc, val) {
        return acc.concat(val);
      }, []);
      return _leaflet2["default"].polygon(shapes, df.get(i));
    });
  }
};

methods.addGeoJSON = function (data, layerId, group, style) {
  // This time, self is actually needed because the callbacks below need
  // to access both the inner and outer senses of "this"
  var self = this;

  if (typeof data === "string") {
    data = JSON.parse(data);
  }

  var globalStyle = _jquery2["default"].extend({}, style, data.style || {});

  var gjlayer = _leaflet2["default"].geoJson(data, {
    style: function style(feature) {
      if (feature.style || feature.properties.style) {
        return _jquery2["default"].extend({}, globalStyle, feature.style, feature.properties.style);
      } else {
        return globalStyle;
      }
    },
    onEachFeature: function onEachFeature(feature, layer) {
      var extraInfo = {
        featureId: feature.id,
        properties: feature.properties
      };
      var popup = feature.properties ? feature.properties.popup : null;
      if (typeof popup !== "undefined" && popup !== null) layer.bindPopup(popup);
      layer.on("click", mouseHandler(self.id, layerId, group, "geojson_click", extraInfo), this);
      layer.on("mouseover", mouseHandler(self.id, layerId, group, "geojson_mouseover", extraInfo), this);
      layer.on("mouseout", mouseHandler(self.id, layerId, group, "geojson_mouseout", extraInfo), this);
    }
  });

  this.layerManager.addLayer(gjlayer, "geojson", layerId, group);
};

methods.removeGeoJSON = function (layerId) {
  this.layerManager.removeLayer("geojson", layerId);
};

methods.clearGeoJSON = function () {
  this.layerManager.clearLayers("geojson");
};

methods.addTopoJSON = function (data, layerId, group, style) {
  // This time, self is actually needed because the callbacks below need
  // to access both the inner and outer senses of "this"
  var self = this;

  if (typeof data === "string") {
    data = JSON.parse(data);
  }

  var globalStyle = _jquery2["default"].extend({}, style, data.style || {});

  var gjlayer = _leaflet2["default"].geoJson(null, {
    style: function style(feature) {
      if (feature.style || feature.properties.style) {
        return _jquery2["default"].extend({}, globalStyle, feature.style, feature.properties.style);
      } else {
        return globalStyle;
      }
    },
    onEachFeature: function onEachFeature(feature, layer) {
      var extraInfo = {
        featureId: feature.id,
        properties: feature.properties
      };
      var popup = feature.properties.popup;
      if (typeof popup !== "undefined" && popup !== null) layer.bindPopup(popup);
      layer.on("click", mouseHandler(self.id, layerId, group, "topojson_click", extraInfo), this);
      layer.on("mouseover", mouseHandler(self.id, layerId, group, "topojson_mouseover", extraInfo), this);
      layer.on("mouseout", mouseHandler(self.id, layerId, group, "topojson_mouseout", extraInfo), this);
    }
  });

  global.omnivore.topojson.parse(data, null, gjlayer);
  this.layerManager.addLayer(gjlayer, "topojson", layerId, group);
};

methods.removeTopoJSON = function (layerId) {
  this.layerManager.removeLayer("topojson", layerId);
};

methods.clearTopoJSON = function () {
  this.layerManager.clearLayers("topojson");
};

methods.addControl = function (html, position, layerId, classes) {
  function onAdd(map) {
    var div = _leaflet2["default"].DomUtil.create("div", classes);

    if (typeof layerId !== "undefined" && layerId !== null) {
      div.setAttribute("id", layerId);
    }

    this._div = div; // It's possible for window.Shiny to be true but Shiny.initializeInputs to
    // not be, when a static leaflet widget is included as part of the shiny
    // UI directly (not through leafletOutput or uiOutput). In this case we
    // don't do the normal Shiny stuff as that will all happen when Shiny
    // itself loads and binds the entire doc.

    if (window.Shiny && _shiny2["default"].initializeInputs) {
      _shiny2["default"].renderHtml(html, this._div);

      _shiny2["default"].initializeInputs(this._div);

      _shiny2["default"].bindAll(this._div);
    } else {
      this._div.innerHTML = html;
    }

    return this._div;
  }

  function onRemove(map) {
    if (window.Shiny && _shiny2["default"].unbindAll) {
      _shiny2["default"].unbindAll(this._div);
    }
  }

  var Control = _leaflet2["default"].Control.extend({
    options: {
      position: position
    },
    onAdd: onAdd,
    onRemove: onRemove
  });

  this.controls.add(new Control(), layerId, html);
};

methods.addCustomControl = function (control, layerId) {
  this.controls.add(control, layerId);
};

methods.removeControl = function (layerId) {
  this.controls.remove(layerId);
};

methods.getControl = function (layerId) {
  this.controls.get(layerId);
};

methods.clearControls = function () {
  this.controls.clear();
};

methods.addLegend = function (options) {
  var legend = _leaflet2["default"].control({
    position: options.position
  });

  var gradSpan;

  legend.onAdd = function (map) {
    var div = _leaflet2["default"].DomUtil.create("div", options.className),
        colors = options.colors,
        labels = options.labels,
        legendHTML = "";

    if (options.type === "numeric") {
      // # Formatting constants.
      var singleBinHeight = 20; // The distance between tick marks, in px

      var vMargin = 8; // If 1st tick mark starts at top of gradient, how
      // many extra px are needed for the top half of the
      // 1st label? (ditto for last tick mark/label)

      var tickWidth = 4; // How wide should tick marks be, in px?

      var labelPadding = 6; // How much distance to reserve for tick mark?
      // (Must be >= tickWidth)
      // # Derived formatting parameters.
      // What's the height of a single bin, in percentage (of gradient height)?
      // It might not just be 1/(n-1), if the gradient extends past the tick
      // marks (which can be the case for pretty cut points).

      var singleBinPct = (options.extra.p_n - options.extra.p_1) / (labels.length - 1); // Each bin is `singleBinHeight` high. How tall is the gradient?

      var totalHeight = 1 / singleBinPct * singleBinHeight + 1; // How far should the first tick be shifted down, relative to the top
      // of the gradient?

      var tickOffset = singleBinHeight / singleBinPct * options.extra.p_1;
      gradSpan = (0, _jquery2["default"])("<span/>").css({
        "background": "linear-gradient(" + colors + ")",
        "opacity": options.opacity,
        "height": totalHeight + "px",
        "width": "18px",
        "display": "block",
        "margin-top": vMargin + "px"
      });
      var leftDiv = (0, _jquery2["default"])("<div/>").css("float", "left"),
          rightDiv = (0, _jquery2["default"])("<div/>").css("float", "left");
      leftDiv.append(gradSpan);
      (0, _jquery2["default"])(div).append(leftDiv).append(rightDiv).append((0, _jquery2["default"])("<br>")); // Have to attach the div to the body at this early point, so that the
      // svg text getComputedTextLength() actually works, below.

      document.body.appendChild(div);
      var ns = "http://www.w3.org/2000/svg";
      var svg = document.createElementNS(ns, "svg");
      rightDiv.append(svg);
      var g = document.createElementNS(ns, "g");
      (0, _jquery2["default"])(g).attr("transform", "translate(0, " + vMargin + ")");
      svg.appendChild(g); // max label width needed to set width of svg, and right-justify text

      var maxLblWidth = 0; // Create tick marks and labels

      _jquery2["default"].each(labels, function (i, label) {
        var y = tickOffset + i * singleBinHeight + 0.5;
        var thisLabel = document.createElementNS(ns, "text");
        (0, _jquery2["default"])(thisLabel).text(labels[i]).attr("y", y).attr("dx", labelPadding).attr("dy", "0.5ex");
        g.appendChild(thisLabel);
        maxLblWidth = Math.max(maxLblWidth, thisLabel.getComputedTextLength());
        var thisTick = document.createElementNS(ns, "line");
        (0, _jquery2["default"])(thisTick).attr("x1", 0).attr("x2", tickWidth).attr("y1", y).attr("y2", y).attr("stroke-width", 1);
        g.appendChild(thisTick);
      }); // Now that we know the max label width, we can right-justify


      (0, _jquery2["default"])(svg).find("text").attr("dx", labelPadding + maxLblWidth).attr("text-anchor", "end"); // Final size for <svg>

      (0, _jquery2["default"])(svg).css({
        width: maxLblWidth + labelPadding + "px",
        height: totalHeight + vMargin * 2 + "px"
      });

      if (options.na_color && _jquery2["default"].inArray(options.na_label, labels) < 0) {
        (0, _jquery2["default"])(div).append("<div><i style=\"" + "background:" + options.na_color + ";opacity:" + options.opacity + ";margin-right:" + labelPadding + "px" + ";\"></i>" + options.na_label + "</div>");
      }
    } else {
      if (options.na_color && _jquery2["default"].inArray(options.na_label, labels) < 0) {
        colors.push(options.na_color);
        labels.push(options.na_label);
      }

      for (var i = 0; i < colors.length; i++) {
        legendHTML += "<i style=\"background:" + colors[i] + ";opacity:" + options.opacity + "\"></i> " + labels[i] + "<br>";
      }

      div.innerHTML = legendHTML;
    }

    if (options.title) (0, _jquery2["default"])(div).prepend("<div style=\"margin-bottom:3px\"><strong>" + options.title + "</strong></div>");
    return div;
  };

  if (options.group) {
    // Auto generate a layerID if not provided
    if (!options.layerId) {
      options.layerId = _leaflet2["default"].Util.stamp(legend);
    }

    var map = this;
    map.on("overlayadd", function (e) {
      if (e.name === options.group) {
        map.controls.add(legend, options.layerId);
      }
    });
    map.on("overlayremove", function (e) {
      if (e.name === options.group) {
        map.controls.remove(options.layerId);
      }
    });
    map.on("groupadd", function (e) {
      if (e.name === options.group) {
        map.controls.add(legend, options.layerId);
      }
    });
    map.on("groupremove", function (e) {
      if (e.name === options.group) {
        map.controls.remove(options.layerId);
      }
    });
  }

  this.controls.add(legend, options.layerId);
};

methods.addLayersControl = function (baseGroups, overlayGroups, options) {
  var _this4 = this;

  // Only allow one layers control at a time
  methods.removeLayersControl.call(this);
  var firstLayer = true;
  var base = {};

  _jquery2["default"].each((0, _util.asArray)(baseGroups), function (i, g) {
    var layer = _this4.layerManager.getLayerGroup(g, true);

    if (layer) {
      base[g] = layer; // Check if >1 base layers are visible; if so, hide all but the first one

      if (_this4.hasLayer(layer)) {
        if (firstLayer) {
          firstLayer = false;
        } else {
          _this4.removeLayer(layer);
        }
      }
    }
  });

  var overlay = {};

  _jquery2["default"].each((0, _util.asArray)(overlayGroups), function (i, g) {
    var layer = _this4.layerManager.getLayerGroup(g, true);

    if (layer) {
      overlay[g] = layer;
    }
  });

  this.currentLayersControl = _leaflet2["default"].control.layers(base, overlay, options);
  this.addControl(this.currentLayersControl);
};

methods.removeLayersControl = function () {
  if (this.currentLayersControl) {
    this.removeControl(this.currentLayersControl);
    this.currentLayersControl = null;
  }
};

methods.addScaleBar = function (options) {
  // Only allow one scale bar at a time
  methods.removeScaleBar.call(this);

  var scaleBar = _leaflet2["default"].control.scale(options).addTo(this);

  this.currentScaleBar = scaleBar;
};

methods.removeScaleBar = function () {
  if (this.currentScaleBar) {
    this.currentScaleBar.remove();
    this.currentScaleBar = null;
  }
};

methods.hideGroup = function (group) {
  var _this5 = this;

  _jquery2["default"].each((0, _util.asArray)(group), function (i, g) {
    var layer = _this5.layerManager.getLayerGroup(g, true);

    if (layer) {
      _this5.removeLayer(layer);
    }
  });
};

methods.showGroup = function (group) {
  var _this6 = this;

  _jquery2["default"].each((0, _util.asArray)(group), function (i, g) {
    var layer = _this6.layerManager.getLayerGroup(g, true);

    if (layer) {
      _this6.addLayer(layer);
    }
  });
};

function setupShowHideGroupsOnZoom(map) {
  if (map.leafletr._hasInitializedShowHideGroups) {
    return;
  }

  map.leafletr._hasInitializedShowHideGroups = true;

  function setVisibility(layer, visible, group) {
    if (visible !== map.hasLayer(layer)) {
      if (visible) {
        map.addLayer(layer);
        map.fire("groupadd", {
          "name": group,
          "layer": layer
        });
      } else {
        map.removeLayer(layer);
        map.fire("groupremove", {
          "name": group,
          "layer": layer
        });
      }
    }
  }

  function showHideGroupsOnZoom() {
    if (!map.layerManager) return;
    var zoom = map.getZoom();
    map.layerManager.getAllGroupNames().forEach(function (group) {
      var layer = map.layerManager.getLayerGroup(group, false);

      if (layer && typeof layer.zoomLevels !== "undefined") {
        setVisibility(layer, layer.zoomLevels === true || layer.zoomLevels.indexOf(zoom) >= 0, group);
      }
    });
  }

  map.showHideGroupsOnZoom = showHideGroupsOnZoom;
  map.on("zoomend", showHideGroupsOnZoom);
}

methods.setGroupOptions = function (group, options) {
  var _this7 = this;

  _jquery2["default"].each((0, _util.asArray)(group), function (i, g) {
    var layer = _this7.layerManager.getLayerGroup(g, true); // This slightly tortured check is because 0 is a valid value for zoomLevels


    if (typeof options.zoomLevels !== "undefined" && options.zoomLevels !== null) {
      layer.zoomLevels = (0, _util.asArray)(options.zoomLevels);
    }
  });

  setupShowHideGroupsOnZoom(this);
  this.showHideGroupsOnZoom();
};

methods.addRasterImage = function (uri, bounds, opacity, attribution, layerId, group) {
  // uri is a data URI containing an image. We want to paint this image as a
  // layer at (top-left) bounds[0] to (bottom-right) bounds[1].
  // We can't simply use ImageOverlay, as it uses bilinear scaling which looks
  // awful as you zoom in (and sometimes shifts positions or disappears).
  // Instead, we'll use a TileLayer.Canvas to draw pieces of the image.
  // First, some helper functions.
  // degree2tile converts latitude, longitude, and zoom to x and y tile
  // numbers. The tile numbers returned can be non-integral, as there's no
  // reason to expect that the lat/lng inputs are exactly on the border of two
  // tiles.
  //
  // We'll use this to convert the bounds we got from the server, into coords
  // in tile-space at a given zoom level. Note that once we do the conversion,
  // we don't to do any more trigonometry to convert between pixel coordinates
  // and tile coordinates; the source image pixel coords, destination canvas
  // pixel coords, and tile coords all can be scaled linearly.
  function degree2tile(lat, lng, zoom) {
    // See http://wiki.openstreetmap.org/wiki/Slippy_map_tilenames
    var latRad = lat * Math.PI / 180;
    var n = Math.pow(2, zoom);
    var x = (lng + 180) / 360 * n;
    var y = (1 - Math.log(Math.tan(latRad) + 1 / Math.cos(latRad)) / Math.PI) / 2 * n;
    return {
      x: x,
      y: y
    };
  } // Given a range [from,to) and either one or two numbers, returns true if
  // there is any overlap between [x,x1) and the range--or if x1 is omitted,
  // then returns true if x is within [from,to).


  function overlap(from, to, x,
  /* optional */
  x1) {
    if (arguments.length == 3) x1 = x;
    return x < to && x1 >= from;
  }

  function getCanvasSmoothingProperty(ctx) {
    var candidates = ["imageSmoothingEnabled", "mozImageSmoothingEnabled", "webkitImageSmoothingEnabled", "msImageSmoothingEnabled"];

    for (var i = 0; i < candidates.length; i++) {
      if (typeof ctx[candidates[i]] !== "undefined") {
        return candidates[i];
      }
    }

    return null;
  } // Our general strategy is to:
  // 1. Load the data URI in an Image() object, so we can get its pixel
  //    dimensions and the underlying image data. (We could have done this
  //    by not encoding as PNG at all but just send an array of RGBA values
  //    from the server, but that would inflate the JSON too much.)
  // 2. Create a hidden canvas that we use just to extract the image data
  //    from the Image (using Context2D.getImageData()).
  // 3. Create a TileLayer.Canvas and add it to the map.
  // We want to synchronously create and attach the TileLayer.Canvas (so an
  // immediate call to clearRasters() will be respected, for example), but
  // Image loads its data asynchronously. Fortunately we can resolve this
  // by putting TileLayer.Canvas into async mode, which will let us create
  // and attach the layer but have it wait until the image is loaded before
  // it actually draws anything.
  // These are the variables that we will populate once the image is loaded.


  var imgData = null; // 1d row-major array, four [0-255] integers per pixel

  var imgDataMipMapper = null;
  var w = null; // image width in pixels

  var h = null; // image height in pixels
  // We'll use this array to store callbacks that need to be invoked once
  // imgData, w, and h have been resolved.

  var imgDataCallbacks = []; // Consumers of imgData, w, and h can call this to be notified when data
  // is available.

  function getImageData(callback) {
    if (imgData != null) {
      // Must not invoke the callback immediately; it's too confusing and
      // fragile to have a function invoke the callback *either* immediately
      // or in the future. Better to be consistent here.
      setTimeout(function () {
        callback(imgData, w, h, imgDataMipMapper);
      }, 0);
    } else {
      imgDataCallbacks.push(callback);
    }
  }

  var img = new Image();

  img.onload = function () {
    // Save size
    w = img.width;
    h = img.height; // Create a dummy canvas to extract the image data

    var imgDataCanvas = document.createElement("canvas");
    imgDataCanvas.width = w;
    imgDataCanvas.height = h;
    imgDataCanvas.style.display = "none";
    document.body.appendChild(imgDataCanvas);
    var imgDataCtx = imgDataCanvas.getContext("2d");
    imgDataCtx.drawImage(img, 0, 0); // Save the image data.

    imgData = imgDataCtx.getImageData(0, 0, w, h).data;
    imgDataMipMapper = new _mipmapper2["default"](img); // Done with the canvas, remove it from the page so it can be gc'd.

    document.body.removeChild(imgDataCanvas); // Alert any getImageData callers who are waiting.

    for (var i = 0; i < imgDataCallbacks.length; i++) {
      imgDataCallbacks[i](imgData, w, h, imgDataMipMapper);
    }

    imgDataCallbacks = [];
  };

  img.src = uri;

  var canvasTiles = _leaflet2["default"].gridLayer({
    opacity: opacity,
    attribution: attribution,
    detectRetina: true,
    async: true
  }); // NOTE: The done() function MUST NOT be invoked until after the current
  // tick; done() looks in Leaflet's tile cache for the current tile, and
  // since it's still being constructed, it won't be found.


  canvasTiles.createTile = function (tilePoint, done) {
    var zoom = tilePoint.z;

    var canvas = _leaflet2["default"].DomUtil.create("canvas");

    var error; // setup tile width and height according to the options

    var size = this.getTileSize();
    canvas.width = size.x;
    canvas.height = size.y;
    getImageData(function (imgData, w, h, mipmapper) {
      try {
        // The Context2D we'll being drawing onto. It's always 256x256.
        var ctx = canvas.getContext("2d"); // Convert our image data's top-left and bottom-right locations into
        // x/y tile coordinates. This is essentially doing a spherical mercator
        // projection, then multiplying by 2^zoom.

        var topLeft = degree2tile(bounds[0][0], bounds[0][1], zoom);
        var bottomRight = degree2tile(bounds[1][0], bounds[1][1], zoom); // The size of the image in x/y tile coordinates.

        var extent = {
          x: bottomRight.x - topLeft.x,
          y: bottomRight.y - topLeft.y
        }; // Short circuit if tile is totally disjoint from image.

        if (!overlap(tilePoint.x, tilePoint.x + 1, topLeft.x, bottomRight.x)) return;
        if (!overlap(tilePoint.y, tilePoint.y + 1, topLeft.y, bottomRight.y)) return; // The linear resolution of the tile we're drawing is always 256px per tile unit.
        // If the linear resolution (in either direction) of the image is less than 256px
        // per tile unit, then use nearest neighbor; otherwise, use the canvas's built-in
        // scaling.

        var imgRes = {
          x: w / extent.x,
          y: h / extent.y
        }; // We can do the actual drawing in one of three ways:
        // - Call drawImage(). This is easy and fast, and results in smooth
        //   interpolation (bilinear?). This is what we want when we are
        //   reducing the image from its native size.
        // - Call drawImage() with imageSmoothingEnabled=false. This is easy
        //   and fast and gives us nearest-neighbor interpolation, which is what
        //   we want when enlarging the image. However, it's unsupported on many
        //   browsers (including QtWebkit).
        // - Do a manual nearest-neighbor interpolation. This is what we'll fall
        //   back to when enlarging, and imageSmoothingEnabled isn't supported.
        //   In theory it's slower, but still pretty fast on my machine, and the
        //   results look the same AFAICT.
        // Is imageSmoothingEnabled supported? If so, we can let canvas do
        // nearest-neighbor interpolation for us.

        var smoothingProperty = getCanvasSmoothingProperty(ctx);

        if (smoothingProperty || imgRes.x >= 256 && imgRes.y >= 256) {
          // Use built-in scaling
          // Turn off anti-aliasing if necessary
          if (smoothingProperty) {
            ctx[smoothingProperty] = imgRes.x >= 256 && imgRes.y >= 256;
          } // Don't necessarily draw with the full-size image; if we're
          // downscaling, use the mipmapper to get a pre-downscaled image
          // (see comments on Mipmapper class for why this matters).


          mipmapper.getBySize(extent.x * 256, extent.y * 256, function (mip) {
            // It's possible that the image will go off the edge of the canvas--
            // that's OK, the canvas should clip appropriately.
            ctx.drawImage(mip, // Convert abs tile coords to rel tile coords, then *256 to convert
            // to rel pixel coords
            (topLeft.x - tilePoint.x) * 256, (topLeft.y - tilePoint.y) * 256, // Always draw the whole thing and let canvas clip; so we can just
            // convert from size in tile coords straight to pixels
            extent.x * 256, extent.y * 256);
          });
        } else {
          // Use manual nearest-neighbor interpolation
          // Calculate the source image pixel coordinates that correspond with
          // the top-left and bottom-right of this tile. (If the source image
          // only partially overlaps the tile, we use max/min to limit the
          // sourceStart/End to only reflect the overlapping portion.)
          var sourceStart = {
            x: Math.max(0, Math.floor((tilePoint.x - topLeft.x) * imgRes.x)),
            y: Math.max(0, Math.floor((tilePoint.y - topLeft.y) * imgRes.y))
          };
          var sourceEnd = {
            x: Math.min(w, Math.ceil((tilePoint.x + 1 - topLeft.x) * imgRes.x)),
            y: Math.min(h, Math.ceil((tilePoint.y + 1 - topLeft.y) * imgRes.y))
          }; // The size, in dest pixels, that each source pixel should occupy.
          // This might be greater or less than 1 (e.g. if x and y resolution
          // are very different).

          var pixelSize = {
            x: 256 / imgRes.x,
            y: 256 / imgRes.y
          }; // For each pixel in the source image that overlaps the tile...

          for (var row = sourceStart.y; row < sourceEnd.y; row++) {
            for (var col = sourceStart.x; col < sourceEnd.x; col++) {
              // ...extract the pixel data...
              var i = (row * w + col) * 4;
              var r = imgData[i];
              var g = imgData[i + 1];
              var b = imgData[i + 2];
              var a = imgData[i + 3];
              ctx.fillStyle = "rgba(" + [r, g, b, a / 255].join(",") + ")"; // ...calculate the corresponding pixel coord in the dest image
              // where it should be drawn...

              var pixelPos = {
                x: (col / imgRes.x + topLeft.x - tilePoint.x) * 256,
                y: (row / imgRes.y + topLeft.y - tilePoint.y) * 256
              }; // ...and draw a rectangle there.

              ctx.fillRect(Math.round(pixelPos.x), Math.round(pixelPos.y), // Looks crazy, but this is necessary to prevent rounding from
              // causing overlap between this rect and its neighbors. The
              // minuend is the location of the next pixel, while the
              // subtrahend is the position of the current pixel (to turn an
              // absolute coordinate to a width/height). Yes, I had to look
              // up minuend and subtrahend.
              Math.round(pixelPos.x + pixelSize.x) - Math.round(pixelPos.x), Math.round(pixelPos.y + pixelSize.y) - Math.round(pixelPos.y));
            }
          }
        }
      } catch (e) {
        error = e;
      } finally {
        done(error, canvas);
      }
    });
    return canvas;
  };

  this.layerManager.addLayer(canvasTiles, "image", layerId, group);
};

methods.removeImage = function (layerId) {
  this.layerManager.removeLayer("image", layerId);
};

methods.clearImages = function () {
  this.layerManager.clearLayers("image");
};

methods.addMeasure = function (options) {
  // if a measureControl already exists, then remove it and
  //   replace with a new one
  methods.removeMeasure.call(this);
  this.measureControl = _leaflet2["default"].control.measure(options);
  this.addControl(this.measureControl);
};

methods.removeMeasure = function () {
  if (this.measureControl) {
    this.removeControl(this.measureControl);
    this.measureControl = null;
  }
};

methods.addSelect = function (ctGroup) {
  var _this8 = this;

  methods.removeSelect.call(this);
  this._selectButton = _leaflet2["default"].easyButton({
    states: [{
      stateName: "select-inactive",
      icon: "ion-qr-scanner",
      title: "Make a selection",
      onClick: function onClick(btn, map) {
        btn.state("select-active");
        _this8._locationFilter = new _leaflet2["default"].LocationFilter2();

        if (ctGroup) {
          var selectionHandle = new global.crosstalk.SelectionHandle(ctGroup);
          selectionHandle.on("change", function (e) {
            if (e.sender !== selectionHandle) {
              if (_this8._locationFilter) {
                _this8._locationFilter.disable();

                btn.state("select-inactive");
              }
            }
          });

          var handler = function handler(e) {
            _this8.layerManager.brush(_this8._locationFilter.getBounds(), {
              sender: selectionHandle
            });
          };

          _this8._locationFilter.on("enabled", handler);

          _this8._locationFilter.on("change", handler);

          _this8._locationFilter.on("disabled", function () {
            selectionHandle.close();
            _this8._locationFilter = null;
          });
        }

        _this8._locationFilter.addTo(map);
      }
    }, {
      stateName: "select-active",
      icon: "ion-close-round",
      title: "Dismiss selection",
      onClick: function onClick(btn, map) {
        btn.state("select-inactive");

        _this8._locationFilter.disable(); // If explicitly dismissed, clear the crosstalk selections


        _this8.layerManager.unbrush();
      }
    }]
  });

  this._selectButton.addTo(this);
};

methods.removeSelect = function () {
  if (this._locationFilter) {
    this._locationFilter.disable();
  }

  if (this._selectButton) {
    this.removeControl(this._selectButton);
    this._selectButton = null;
  }
};

methods.createMapPane = function (name, zIndex) {
  this.createPane(name);
  this.getPane(name).style.zIndex = zIndex;
};


}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{"./cluster-layer-store":1,"./crs_utils":3,"./dataframe":4,"./global/htmlwidgets":8,"./global/jquery":9,"./global/leaflet":10,"./global/shiny":12,"./mipmapper":16,"./util":17}],16:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

// This class simulates a mipmap, which shrinks images by powers of two. This
// stepwise reduction results in "pixel-perfect downscaling" (where every
// pixel of the original image has some contribution to the downscaled image)
// as opposed to a single-step downscaling which will discard a lot of data
// (and with sparse images at small scales can give very surprising results).
var Mipmapper = /*#__PURE__*/function () {
  function Mipmapper(img) {
    _classCallCheck(this, Mipmapper);

    this._layers = [img];
  } // The various functions on this class take a callback function BUT MAY OR MAY
  // NOT actually behave asynchronously.


  _createClass(Mipmapper, [{
    key: "getBySize",
    value: function getBySize(desiredWidth, desiredHeight, callback) {
      var _this = this;

      var i = 0;
      var lastImg = this._layers[0];

      var testNext = function testNext() {
        _this.getByIndex(i, function (img) {
          // If current image is invalid (i.e. too small to be rendered) or
          // it's smaller than what we wanted, return the last known good image.
          if (!img || img.width < desiredWidth || img.height < desiredHeight) {
            callback(lastImg);
            return;
          } else {
            lastImg = img;
            i++;
            testNext();
            return;
          }
        });
      };

      testNext();
    }
  }, {
    key: "getByIndex",
    value: function getByIndex(i, callback) {
      var _this2 = this;

      if (this._layers[i]) {
        callback(this._layers[i]);
        return;
      }

      this.getByIndex(i - 1, function (prevImg) {
        if (!prevImg) {
          // prevImg could not be calculated (too small, possibly)
          callback(null);
          return;
        }

        if (prevImg.width < 2 || prevImg.height < 2) {
          // Can't reduce this image any further
          callback(null);
          return;
        } // If reduce ever becomes truly asynchronous, we should stuff a promise or
        // something into this._layers[i] before calling this.reduce(), to prevent
        // redundant reduce operations from happening.


        _this2.reduce(prevImg, function (reducedImg) {
          _this2._layers[i] = reducedImg;
          callback(reducedImg);
          return;
        });
      });
    }
  }, {
    key: "reduce",
    value: function reduce(img, callback) {
      var imgDataCanvas = document.createElement("canvas");
      imgDataCanvas.width = Math.ceil(img.width / 2);
      imgDataCanvas.height = Math.ceil(img.height / 2);
      imgDataCanvas.style.display = "none";
      document.body.appendChild(imgDataCanvas);

      try {
        var imgDataCtx = imgDataCanvas.getContext("2d");
        imgDataCtx.drawImage(img, 0, 0, img.width / 2, img.height / 2);
        callback(imgDataCanvas);
      } finally {
        document.body.removeChild(imgDataCanvas);
      }
    }
  }]);

  return Mipmapper;
}();

exports["default"] = Mipmapper;


},{}],17:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.log = log;
exports.recycle = recycle;
exports.asArray = asArray;

function log(message) {
  /* eslint-disable no-console */
  if (console && console.log) console.log(message);
  /* eslint-enable no-console */
}

function recycle(values, length, inPlace) {
  if (length === 0 && !inPlace) return [];

  if (!(values instanceof Array)) {
    if (inPlace) {
      throw new Error("Can't do in-place recycling of a non-Array value");
    }

    values = [values];
  }

  if (typeof length === "undefined") length = values.length;
  var dest = inPlace ? values : [];
  var origLength = values.length;

  while (dest.length < length) {
    dest.push(values[dest.length % origLength]);
  }

  if (dest.length > length) {
    dest.splice(length, dest.length - length);
  }

  return dest;
}

function asArray(value) {
  if (value instanceof Array) return value;else return [value];
}


},{}]},{},[13]);
</script>
<script>(function (root, factory) {
	if (typeof define === 'function' && define.amd) {
		// AMD. Register as an anonymous module.
		define(['leaflet'], factory);
	} else if (typeof modules === 'object' && module.exports) {
		// define a Common JS module that relies on 'leaflet'
		module.exports = factory(require('leaflet'));
	} else {
		// Assume Leaflet is loaded into global object L already
		factory(L);
	}
}(this, function (L) {
	'use strict';

	L.TileLayer.Provider = L.TileLayer.extend({
		initialize: function (arg, options) {
			var providers = L.TileLayer.Provider.providers;

			var parts = arg.split('.');

			var providerName = parts[0];
			var variantName = parts[1];

			if (!providers[providerName]) {
				throw 'No such provider (' + providerName + ')';
			}

			var provider = {
				url: providers[providerName].url,
				options: providers[providerName].options
			};

			// overwrite values in provider from variant.
			if (variantName && 'variants' in providers[providerName]) {
				if (!(variantName in providers[providerName].variants)) {
					throw 'No such variant of ' + providerName + ' (' + variantName + ')';
				}
				var variant = providers[providerName].variants[variantName];
				var variantOptions;
				if (typeof variant === 'string') {
					variantOptions = {
						variant: variant
					};
				} else {
					variantOptions = variant.options;
				}
				provider = {
					url: variant.url || provider.url,
					options: L.Util.extend({}, provider.options, variantOptions)
				};
			}

			// replace attribution placeholders with their values from toplevel provider attribution,
			// recursively
			var attributionReplacer = function (attr) {
				if (attr.indexOf('{attribution.') === -1) {
					return attr;
				}
				return attr.replace(/\{attribution.(\w*)\}/g,
					function (match, attributionName) {
						return attributionReplacer(providers[attributionName].options.attribution);
					}
				);
			};
			provider.options.attribution = attributionReplacer(provider.options.attribution);

			// Compute final options combining provider options with any user overrides
			var layerOpts = L.Util.extend({}, provider.options, options);
			L.TileLayer.prototype.initialize.call(this, provider.url, layerOpts);
		}
	});

	/**
	 * Definition of providers.
	 * see http://leafletjs.com/reference.html#tilelayer for options in the options map.
	 */

	L.TileLayer.Provider.providers = {
		OpenStreetMap: {
			url: '//{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
			options: {
				maxZoom: 19,
				attribution:
					'&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
			},
			variants: {
				Mapnik: {},
				DE: {
					url: '//{s}.tile.openstreetmap.de/tiles/osmde/{z}/{x}/{y}.png',
					options: {
						maxZoom: 18
					}
				},
				CH: {
					url: '//tile.osm.ch/switzerland/{z}/{x}/{y}.png',
					options: {
						maxZoom: 18,
						bounds: [[45, 5], [48, 11]]
					}
				},
				France: {
					url: '//{s}.tile.openstreetmap.fr/osmfr/{z}/{x}/{y}.png',
					options: {
						maxZoom: 20,
						attribution: '&copy; Openstreetmap France | {attribution.OpenStreetMap}'
					}
				},
				HOT: {
					url: '//{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png',
					options: {
						attribution:
							'{attribution.OpenStreetMap}, ' +
							'Tiles style by <a href="https://www.hotosm.org/" target="_blank">Humanitarian OpenStreetMap Team</a> ' +
							'hosted by <a href="https://openstreetmap.fr/" target="_blank">OpenStreetMap France</a>'
					}
				},
				BZH: {
					url: '//tile.openstreetmap.bzh/br/{z}/{x}/{y}.png',
					options: {
						attribution: '{attribution.OpenStreetMap}, Tiles courtesy of <a href="http://www.openstreetmap.bzh/" target="_blank">Breton OpenStreetMap Team</a>',
						bounds: [[46.2, -5.5], [50, 0.7]]
					}
				}
			}
		},
		OpenSeaMap: {
			url: '//tiles.openseamap.org/seamark/{z}/{x}/{y}.png',
			options: {
				attribution: 'Map data: &copy; <a href="http://www.openseamap.org">OpenSeaMap</a> contributors'
			}
		},
		OpenPtMap: {
			url: 'http://openptmap.org/tiles/{z}/{x}/{y}.png',
			options: {
				maxZoom: 17,
				attribution: 'Map data: &copy; <a href="http://www.openptmap.org">OpenPtMap</a> contributors'
			}
		},
		OpenTopoMap: {
			url: 'https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png',
			options: {
				maxZoom: 17,
				attribution: 'Map data: {attribution.OpenStreetMap}, <a href="http://viewfinderpanoramas.org">SRTM</a> | Map style: &copy; <a href="https://opentopomap.org">OpenTopoMap</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)'
			}
		},
		OpenRailwayMap: {
			url: 'https://{s}.tiles.openrailwaymap.org/standard/{z}/{x}/{y}.png',
			options: {
				maxZoom: 19,
				attribution: 'Map data: {attribution.OpenStreetMap} | Map style: &copy; <a href="https://www.OpenRailwayMap.org">OpenRailwayMap</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)'
			}
		},
		OpenFireMap: {
			url: 'http://openfiremap.org/hytiles/{z}/{x}/{y}.png',
			options: {
				maxZoom: 19,
				attribution: 'Map data: {attribution.OpenStreetMap} | Map style: &copy; <a href="http://www.openfiremap.org">OpenFireMap</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)'
			}
		},
		SafeCast: {
			url: '//s3.amazonaws.com/te512.safecast.org/{z}/{x}/{y}.png',
			options: {
				maxZoom: 16,
				attribution: 'Map data: {attribution.OpenStreetMap} | Map style: &copy; <a href="https://blog.safecast.org/about/">SafeCast</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)'
			}
		},
		Thunderforest: {
			url: 'https://{s}.tile.thunderforest.com/{variant}/{z}/{x}/{y}.png?apikey={apikey}',
			options: {
				attribution:
					'&copy; <a href="http://www.thunderforest.com/">Thunderforest</a>, {attribution.OpenStreetMap}',
				variant: 'cycle',
				apikey: '<insert your api key here>',
				maxZoom: 22
			},
			variants: {
				OpenCycleMap: 'cycle',
				Transport: {
					options: {
						variant: 'transport'
					}
				},
				TransportDark: {
					options: {
						variant: 'transport-dark'
					}
				},
				SpinalMap: {
					options: {
						variant: 'spinal-map'
					}
				},
				Landscape: 'landscape',
				Outdoors: 'outdoors',
				Pioneer: 'pioneer',
				MobileAtlas: 'mobile-atlas',
				Neighbourhood: 'neighbourhood'
			}
		},
		OpenMapSurfer: {
			url: 'https://maps.heigit.org/openmapsurfer/tiles/{variant}/webmercator/{z}/{x}/{y}.png',
			options: {
				maxZoom: 19,
				variant: 'roads',
				attribution: 'Imagery from <a href="http://giscience.uni-hd.de/">GIScience Research Group @ University of Heidelberg</a> | Map data '
			},
			variants: {
				Roads: {
					options: {
						variant: 'roads',
						attribution: '{attribution.OpenMapSurfer}{attribution.OpenStreetMap}'
					}
				},
				Hybrid: {
					options: {
						variant: 'hybrid',
						attribution: '{attribution.OpenMapSurfer}{attribution.OpenStreetMap}'
					}
				},
				AdminBounds: {
					options: {
						variant: 'adminb',
						maxZoom: 18,
						attribution: '{attribution.OpenMapSurfer}{attribution.OpenStreetMap}'
					}
				},
				ContourLines: {
					options: {
						variant: 'asterc',
						maxZoom: 18,
						minZoom: 13,
						attribution: '{attribution.OpenMapSurfer} <a href="https://lpdaac.usgs.gov/products/aster_policies">ASTER GDEM</a>'
					}
				},
				Hillshade: {
					options: {
						variant: 'asterh',
						maxZoom: 18,
						attribution: '{attribution.OpenMapSurfer} <a href="https://lpdaac.usgs.gov/products/aster_policies">ASTER GDEM</a>, <a href="http://srtm.csi.cgiar.org/">SRTM</a>'
					}
				},
				ElementsAtRisk: {
					options: {
						variant: 'elements_at_risk',
						attribution: '{attribution.OpenMapSurfer}{attribution.OpenStreetMap}'
					}
				}
			}
		},
		Hydda: {
			url: '//{s}.tile.openstreetmap.se/hydda/{variant}/{z}/{x}/{y}.png',
			options: {
				maxZoom: 18,
				variant: 'full',
				attribution: 'Tiles courtesy of <a href="http://openstreetmap.se/" target="_blank">OpenStreetMap Sweden</a> &mdash; Map data {attribution.OpenStreetMap}'
			},
			variants: {
				Full: 'full',
				Base: 'base',
				RoadsAndLabels: 'roads_and_labels'
			}
		},
		MapBox: {
			url: 'https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}{r}.png?access_token={accessToken}',
			options: {
				attribution:
					'<a href="https://www.mapbox.com/about/maps/" target="_blank">&copy; Mapbox</a> ' +
					'{attribution.OpenStreetMap} ' +
					'<a href="https://www.mapbox.com/map-feedback/" target="_blank">Improve this map</a>',
				subdomains: 'abcd',
				id: 'mapbox.streets',
				accessToken: '<insert your access token here>',
			}
		},
		Stamen: {
			url: '//stamen-tiles-{s}.a.ssl.fastly.net/{variant}/{z}/{x}/{y}{r}.{ext}',
			options: {
				attribution:
					'Map tiles by <a href="http://stamen.com">Stamen Design</a>, ' +
					'<a href="http://creativecommons.org/licenses/by/3.0">CC BY 3.0</a> &mdash; ' +
					'Map data {attribution.OpenStreetMap}',
				subdomains: 'abcd',
				minZoom: 0,
				maxZoom: 20,
				variant: 'toner',
				ext: 'png'
			},
			variants: {
				Toner: 'toner',
				TonerBackground: 'toner-background',
				TonerHybrid: 'toner-hybrid',
				TonerLines: 'toner-lines',
				TonerLabels: 'toner-labels',
				TonerLite: 'toner-lite',
				Watercolor: {
					url: '//stamen-tiles-{s}.a.ssl.fastly.net/{variant}/{z}/{x}/{y}.{ext}',
					options: {
						variant: 'watercolor',
						ext: 'jpg',
						minZoom: 1,
						maxZoom: 16
					}
				},
				Terrain: {
					options: {
						variant: 'terrain',
						minZoom: 0,
						maxZoom: 18
					}
				},
				TerrainBackground: {
					options: {
						variant: 'terrain-background',
						minZoom: 0,
						maxZoom: 18
					}
				},
				TerrainLabels: {
					options: {
						variant: 'terrain-labels',
						minZoom: 0,
						maxZoom: 18
					}
				},
				TopOSMRelief: {
					url: '//stamen-tiles-{s}.a.ssl.fastly.net/{variant}/{z}/{x}/{y}.{ext}',
					options: {
						variant: 'toposm-color-relief',
						ext: 'jpg',
						bounds: [[22, -132], [51, -56]]
					}
				},
				TopOSMFeatures: {
					options: {
						variant: 'toposm-features',
						bounds: [[22, -132], [51, -56]],
						opacity: 0.9
					}
				}
			}
		},
		TomTom: {
			url: 'https://{s}.api.tomtom.com/map/1/tile/{variant}/{style}/{z}/{x}/{y}.{ext}?key={apikey}',
			options: {
				variant: 'basic',
				maxZoom: 22,
				attribution:
					'<a href="https://tomtom.com" target="_blank">&copy;  1992 - ' + new Date().getFullYear() + ' TomTom.</a> ',
				subdomains: 'abcd',
				style: 'main',
				ext: 'png',
				apikey: '<insert your API key here>',
			},
			variants: {
				Basic: 'basic',
				Hybrid: 'hybrid',
				Labels: 'labels'
			}
		},
		Esri: {
			url: '//server.arcgisonline.com/ArcGIS/rest/services/{variant}/MapServer/tile/{z}/{y}/{x}',
			options: {
				variant: 'World_Street_Map',
				attribution: 'Tiles &copy; Esri'
			},
			variants: {
				WorldStreetMap: {
					options: {
						attribution:
							'{attribution.Esri} &mdash; ' +
							'Source: Esri, DeLorme, NAVTEQ, USGS, Intermap, iPC, NRCAN, Esri Japan, METI, Esri China (Hong Kong), Esri (Thailand), TomTom, 2012'
					}
				},
				DeLorme: {
					options: {
						variant: 'Specialty/DeLorme_World_Base_Map',
						minZoom: 1,
						maxZoom: 11,
						attribution: '{attribution.Esri} &mdash; Copyright: &copy;2012 DeLorme'
					}
				},
				WorldTopoMap: {
					options: {
						variant: 'World_Topo_Map',
						attribution:
							'{attribution.Esri} &mdash; ' +
							'Esri, DeLorme, NAVTEQ, TomTom, Intermap, iPC, USGS, FAO, NPS, NRCAN, GeoBase, Kadaster NL, Ordnance Survey, Esri Japan, METI, Esri China (Hong Kong), and the GIS User Community'
					}
				},
				WorldImagery: {
					options: {
						variant: 'World_Imagery',
						attribution:
							'{attribution.Esri} &mdash; ' +
							'Source: Esri, i-cubed, USDA, USGS, AEX, GeoEye, Getmapping, Aerogrid, IGN, IGP, UPR-EGP, and the GIS User Community'
					}
				},
				WorldTerrain: {
					options: {
						variant: 'World_Terrain_Base',
						maxZoom: 13,
						attribution:
							'{attribution.Esri} &mdash; ' +
							'Source: USGS, Esri, TANA, DeLorme, and NPS'
					}
				},
				WorldShadedRelief: {
					options: {
						variant: 'World_Shaded_Relief',
						maxZoom: 13,
						attribution: '{attribution.Esri} &mdash; Source: Esri'
					}
				},
				WorldPhysical: {
					options: {
						variant: 'World_Physical_Map',
						maxZoom: 8,
						attribution: '{attribution.Esri} &mdash; Source: US National Park Service'
					}
				},
				OceanBasemap: {
					options: {
						variant: 'Ocean_Basemap',
						maxZoom: 13,
						attribution: '{attribution.Esri} &mdash; Sources: GEBCO, NOAA, CHS, OSU, UNH, CSUMB, National Geographic, DeLorme, NAVTEQ, and Esri'
					}
				},
				NatGeoWorldMap: {
					options: {
						variant: 'NatGeo_World_Map',
						maxZoom: 16,
						attribution: '{attribution.Esri} &mdash; National Geographic, Esri, DeLorme, NAVTEQ, UNEP-WCMC, USGS, NASA, ESA, METI, NRCAN, GEBCO, NOAA, iPC'
					}
				},
				WorldGrayCanvas: {
					options: {
						variant: 'Canvas/World_Light_Gray_Base',
						maxZoom: 16,
						attribution: '{attribution.Esri} &mdash; Esri, DeLorme, NAVTEQ'
					}
				}
			}
		},
		OpenWeatherMap: {
			url: 'http://{s}.tile.openweathermap.org/map/{variant}/{z}/{x}/{y}.png?appid={apiKey}',
			options: {
				maxZoom: 19,
				attribution: 'Map data &copy; <a href="http://openweathermap.org">OpenWeatherMap</a>',
				apiKey:'<insert your api key here>',
				opacity: 0.5
			},
			variants: {
				Clouds: 'clouds',
				CloudsClassic: 'clouds_cls',
				Precipitation: 'precipitation',
				PrecipitationClassic: 'precipitation_cls',
				Rain: 'rain',
				RainClassic: 'rain_cls',
				Pressure: 'pressure',
				PressureContour: 'pressure_cntr',
				Wind: 'wind',
				Temperature: 'temp',
				Snow: 'snow'
			}
		},
		HERE: {
			/*
			 * HERE maps, formerly Nokia maps.
			 * These basemaps are free, but you need an API key. Please sign up at
			 * https://developer.here.com/plans
			 */
			url:
				'https://{s}.{base}.maps.api.here.com/maptile/2.1/' +
				'{type}/{mapID}/{variant}/{z}/{x}/{y}/{size}/{format}?' +
				'app_id={app_id}&app_code={app_code}&lg={language}',
			options: {
				attribution:
					'Map &copy; 1987-' + new Date().getFullYear() + ' <a href="http://developer.here.com">HERE</a>',
				subdomains: '1234',
				mapID: 'newest',
				'app_id': '<insert your app_id here>',
				'app_code': '<insert your app_code here>',
				base: 'base',
				variant: 'normal.day',
				maxZoom: 20,
				type: 'maptile',
				language: 'eng',
				format: 'png8',
				size: '256'
			},
			variants: {
				normalDay: 'normal.day',
				normalDayCustom: 'normal.day.custom',
				normalDayGrey: 'normal.day.grey',
				normalDayMobile: 'normal.day.mobile',
				normalDayGreyMobile: 'normal.day.grey.mobile',
				normalDayTransit: 'normal.day.transit',
				normalDayTransitMobile: 'normal.day.transit.mobile',
				normalDayTraffic: {
					options: {
						variant: 'normal.traffic.day',
						base: 'traffic',
						type: 'traffictile'
					}
				},
				normalNight: 'normal.night',
				normalNightMobile: 'normal.night.mobile',
				normalNightGrey: 'normal.night.grey',
				normalNightGreyMobile: 'normal.night.grey.mobile',
				normalNightTransit: 'normal.night.transit',
				normalNightTransitMobile: 'normal.night.transit.mobile',
				reducedDay: 'reduced.day',
				reducedNight: 'reduced.night',
				basicMap: {
					options: {
						type: 'basetile'
					}
				},
				mapLabels: {
					options: {
						type: 'labeltile',
						format: 'png'
					}
				},
				trafficFlow: {
					options: {
						base: 'traffic',
						type: 'flowtile'
					}
				},
				carnavDayGrey: 'carnav.day.grey',
				hybridDay: {
					options: {
						base: 'aerial',
						variant: 'hybrid.day'
					}
				},
				hybridDayMobile: {
					options: {
						base: 'aerial',
						variant: 'hybrid.day.mobile'
					}
				},
				hybridDayTransit: {
					options: {
						base: 'aerial',
						variant: 'hybrid.day.transit'
					}
				},
				hybridDayGrey: {
					options: {
						base: 'aerial',
						variant: 'hybrid.grey.day'
					}
				},
				hybridDayTraffic: {
					options: {
						variant: 'hybrid.traffic.day',
						base: 'traffic',
						type: 'traffictile'
					}
				},
				pedestrianDay: 'pedestrian.day',
				pedestrianNight: 'pedestrian.night',
				satelliteDay: {
					options: {
						base: 'aerial',
						variant: 'satellite.day'
					}
				},
				terrainDay: {
					options: {
						base: 'aerial',
						variant: 'terrain.day'
					}
				},
				terrainDayMobile: {
					options: {
						base: 'aerial',
						variant: 'terrain.day.mobile'
					}
				}
			}
		},
		FreeMapSK: {
			url: 'http://t{s}.freemap.sk/T/{z}/{x}/{y}.jpeg',
			options: {
				minZoom: 8,
				maxZoom: 16,
				subdomains: '1234',
				bounds: [[47.204642, 15.996093], [49.830896, 22.576904]],
				attribution:
					'{attribution.OpenStreetMap}, vizualization CC-By-SA 2.0 <a href="http://freemap.sk">Freemap.sk</a>'
			}
		},
		MtbMap: {
			url: 'http://tile.mtbmap.cz/mtbmap_tiles/{z}/{x}/{y}.png',
			options: {
				attribution:
					'{attribution.OpenStreetMap} &amp; USGS'
			}
		},
		CartoDB: {
			url: 'https://{s}.basemaps.cartocdn.com/{variant}/{z}/{x}/{y}{r}.png',
			options: {
				attribution: '{attribution.OpenStreetMap} &copy; <a href="https://carto.com/attributions">CARTO</a>',
				subdomains: 'abcd',
				maxZoom: 19,
				variant: 'light_all'
			},
			variants: {
				Positron: 'light_all',
				PositronNoLabels: 'light_nolabels',
				PositronOnlyLabels: 'light_only_labels',
				DarkMatter: 'dark_all',
				DarkMatterNoLabels: 'dark_nolabels',
				DarkMatterOnlyLabels: 'dark_only_labels',
				Voyager: 'rastertiles/voyager',
				VoyagerNoLabels: 'rastertiles/voyager_nolabels',
				VoyagerOnlyLabels: 'rastertiles/voyager_only_labels',
				VoyagerLabelsUnder: 'rastertiles/voyager_labels_under'
			}
		},
		HikeBike: {
			url: 'https://tiles.wmflabs.org/{variant}/{z}/{x}/{y}.png',
			options: {
				maxZoom: 19,
				attribution: '{attribution.OpenStreetMap}',
				variant: 'hikebike'
			},
			variants: {
				HikeBike: {},
				HillShading: {
					options: {
						maxZoom: 15,
						variant: 'hillshading'
					}
				}
			}
		},
		BasemapAT: {
			url: '//maps{s}.wien.gv.at/basemap/{variant}/normal/google3857/{z}/{y}/{x}.{format}',
			options: {
				maxZoom: 19,
				attribution: 'Datenquelle: <a href="https://www.basemap.at">basemap.at</a>',
				subdomains: ['', '1', '2', '3', '4'],
				format: 'png',
				bounds: [[46.358770, 8.782379], [49.037872, 17.189532]],
				variant: 'geolandbasemap'
			},
			variants: {
				basemap: {
					options: {
						maxZoom: 20, // currently only in Vienna
						variant: 'geolandbasemap'
					}
				},
				grau: 'bmapgrau',
				overlay: 'bmapoverlay',
				highdpi: {
					options: {
						variant: 'bmaphidpi',
						format: 'jpeg'
					}
				},
				orthofoto: {
					options: {
						maxZoom: 20, // currently only in Vienna
						variant: 'bmaporthofoto30cm',
						format: 'jpeg'
					}
				}
			}
		},
		nlmaps: {
			url: '//geodata.nationaalgeoregister.nl/tiles/service/wmts/{variant}/EPSG:3857/{z}/{x}/{y}.png',
			options: {
				minZoom: 6,
				maxZoom: 19,
				bounds: [[50.5, 3.25], [54, 7.6]],
				attribution: 'Kaartgegevens &copy; <a href="kadaster.nl">Kadaster</a>'
			},
			variants: {
				'standaard': 'brtachtergrondkaart',
				'pastel': 'brtachtergrondkaartpastel',
				'grijs': 'brtachtergrondkaartgrijs',
				'luchtfoto': {
					'url': '//geodata.nationaalgeoregister.nl/luchtfoto/rgb/wmts/1.0.0/2016_ortho25/EPSG:3857/{z}/{x}/{y}.png',
				}
			}
		},
		NASAGIBS: {
			url: 'https://map1.vis.earthdata.nasa.gov/wmts-webmerc/{variant}/default/{time}/{tilematrixset}{maxZoom}/{z}/{y}/{x}.{format}',
			options: {
				attribution:
					'Imagery provided by services from the Global Imagery Browse Services (GIBS), operated by the NASA/GSFC/Earth Science Data and Information System ' +
					'(<a href="https://earthdata.nasa.gov">ESDIS</a>) with funding provided by NASA/HQ.',
				bounds: [[-85.0511287776, -179.999999975], [85.0511287776, 179.999999975]],
				minZoom: 1,
				maxZoom: 9,
				format: 'jpg',
				time: '',
				tilematrixset: 'GoogleMapsCompatible_Level'
			},
			variants: {
				ModisTerraTrueColorCR: 'MODIS_Terra_CorrectedReflectance_TrueColor',
				ModisTerraBands367CR: 'MODIS_Terra_CorrectedReflectance_Bands367',
				ViirsEarthAtNight2012: {
					options: {
						variant: 'VIIRS_CityLights_2012',
						maxZoom: 8
					}
				},
				ModisTerraLSTDay: {
					options: {
						variant: 'MODIS_Terra_Land_Surface_Temp_Day',
						format: 'png',
						maxZoom: 7,
						opacity: 0.75
					}
				},
				ModisTerraSnowCover: {
					options: {
						variant: 'MODIS_Terra_Snow_Cover',
						format: 'png',
						maxZoom: 8,
						opacity: 0.75
					}
				},
				ModisTerraAOD: {
					options: {
						variant: 'MODIS_Terra_Aerosol',
						format: 'png',
						maxZoom: 6,
						opacity: 0.75
					}
				},
				ModisTerraChlorophyll: {
					options: {
						variant: 'MODIS_Terra_Chlorophyll_A',
						format: 'png',
						maxZoom: 7,
						opacity: 0.75
					}
				}
			}
		},
		NLS: {
			// NLS maps are copyright National library of Scotland.
			// http://maps.nls.uk/projects/api/index.html
			// Please contact NLS for anything other than non-commercial low volume usage
			//
			// Map sources: Ordnance Survey 1:1m to 1:63K, 1920s-1940s
			//   z0-9  - 1:1m
			//  z10-11 - quarter inch (1:253440)
			//  z12-18 - one inch (1:63360)
			url: '//nls-{s}.tileserver.com/nls/{z}/{x}/{y}.jpg',
			options: {
				attribution: '<a href="http://geo.nls.uk/maps/">National Library of Scotland Historic Maps</a>',
				bounds: [[49.6, -12], [61.7, 3]],
				minZoom: 1,
				maxZoom: 18,
				subdomains: '0123',
			}
		},
		JusticeMap: {
			// Justice Map (http://www.justicemap.org/)
			// Visualize race and income data for your community, county and country.
			// Includes tools for data journalists, bloggers and community activists.
			url: 'http://www.justicemap.org/tile/{size}/{variant}/{z}/{x}/{y}.png',
			options: {
				attribution: '<a href="http://www.justicemap.org/terms.php">Justice Map</a>',
				// one of 'county', 'tract', 'block'
				size: 'county',
				// Bounds for USA, including Alaska and Hawaii
				bounds: [[14, -180], [72, -56]]
			},
			variants: {
				income: 'income',
				americanIndian: 'indian',
				asian: 'asian',
				black: 'black',
				hispanic: 'hispanic',
				multi: 'multi',
				nonWhite: 'nonwhite',
				white: 'white',
				plurality: 'plural'
			}
		},
		Wikimedia: {
			url: 'https://maps.wikimedia.org/osm-intl/{z}/{x}/{y}{r}.png',
			options: {
				attribution: '<a href="https://wikimediafoundation.org/wiki/Maps_Terms_of_Use">Wikimedia</a>',
				minZoom: 1,
				maxZoom: 19
			}
		},
		GeoportailFrance: {
			url: 'https://wxs.ign.fr/{apikey}/geoportail/wmts?REQUEST=GetTile&SERVICE=WMTS&VERSION=1.0.0&STYLE={style}&TILEMATRIXSET=PM&FORMAT={format}&LAYER={variant}&TILEMATRIX={z}&TILEROW={y}&TILECOL={x}',
			options: {
				attribution: '<a target="_blank" href="https://www.geoportail.gouv.fr/">Geoportail France</a>',
				bounds: [[-75, -180], [81, 180]],
				minZoom: 2,
				maxZoom: 18,
				// Get your own geoportail apikey here : http://professionnels.ign.fr/ign/contrats/
				// NB : 'choisirgeoportail' is a demonstration key that comes with no guarantee
				apikey: 'choisirgeoportail',
				format: 'image/jpeg',
				style : 'normal',
				variant: 'GEOGRAPHICALGRIDSYSTEMS.MAPS.SCAN-EXPRESS.STANDARD'
			},
			variants: {
				parcels: {
					options : {
						variant: 'CADASTRALPARCELS.PARCELS',
						maxZoom: 20,
						style : 'bdparcellaire',
						format: 'image/png'
					}
				},
				ignMaps: 'GEOGRAPHICALGRIDSYSTEMS.MAPS',
				maps: 'GEOGRAPHICALGRIDSYSTEMS.MAPS.SCAN-EXPRESS.STANDARD',
				orthos: {
					options: {
						maxZoom: 19,
						variant: 'ORTHOIMAGERY.ORTHOPHOTOS'
					}
				}
			}
		},
		OneMapSG: {
			url: '//maps-{s}.onemap.sg/v3/{variant}/{z}/{x}/{y}.png',
			options: {
				variant: 'Default',
				minZoom: 11,
				maxZoom: 18,
				bounds: [[1.56073, 104.11475], [1.16, 103.502]],
				attribution: '<img src="https://docs.onemap.sg/maps/images/oneMap64-01.png" style="height:20px;width:20px;"/> New OneMap | Map data &copy; contributors, <a href="http://SLA.gov.sg">Singapore Land Authority</a>'
			},
			variants: {
				Default: 'Default',
				Night: 'Night',
				Original: 'Original',
				Grey: 'Grey',
				LandLot: 'LandLot'
			}
		}
	};

	L.tileLayer.provider = function (provider, options) {
		return new L.TileLayer.Provider(provider, options);
	};

	return L;
}));
</script>
<script>LeafletWidget.methods.addProviderTiles = function(provider, layerId, group, options) {
  this.layerManager.addLayer(L.tileLayer.provider(provider, options), "tile", layerId, group);
};
</script>
<style type="text/css">.leaflet-control-minimap{border:rgba(255,255,255,1) solid;box-shadow:0 1px 5px rgba(0,0,0,.65);border-radius:3px;background:#f8f8f9;transition:all .6s}.leaflet-control-minimap a{background-color:rgba(255,255,255,1);background-repeat:no-repeat;z-index:99999;transition:all .6s}.leaflet-control-minimap a.minimized-bottomright{-webkit-transform:rotate(180deg);transform:rotate(180deg);border-radius:0}.leaflet-control-minimap a.minimized-topleft{-webkit-transform:rotate(0deg);transform:rotate(0deg);border-radius:0}.leaflet-control-minimap a.minimized-bottomleft{-webkit-transform:rotate(270deg);transform:rotate(270deg);border-radius:0}.leaflet-control-minimap a.minimized-topright{-webkit-transform:rotate(90deg);transform:rotate(90deg);border-radius:0}.leaflet-control-minimap-toggle-display{background-image:url(data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIGhlaWdodD0iMTgiIHdpZHRoPSIxOCI+PGRlZnM+PG1hcmtlciBvcmllbnQ9ImF1dG8iIG92ZXJmbG93PSJ2aXNpYmxlIj48cGF0aCBkPSJNLTIuNi0yLjgyOEwtNS40MjggMC0yLjYgMi44MjguMjI4IDAtMi42LTIuODI4eiIgZmlsbC1ydWxlPSJldmVub2RkIiBzdHJva2U9IiMwMDAiIHN0cm9rZS13aWR0aD0iLjRwdCIvPjwvbWFya2VyPjxtYXJrZXIgb3JpZW50PSJhdXRvIiBvdmVyZmxvdz0idmlzaWJsZSI+PGcgZmlsbD0ibm9uZSIgc3Ryb2tlPSIjMDAwIiBzdHJva2Utd2lkdGg9Ii44IiBzdHJva2UtbGluZWNhcD0icm91bmQiPjxwYXRoIGQ9Ik00LjU2NiA0Ljc1TC0uNjUyIDAiLz48cGF0aCBkPSJNMS41NDQgNC43NUwtMy42NzQgMCIvPjxwYXRoIGQ9Ik0tMS41NjYgNC43NUwtNi43ODQgMCIvPjxwYXRoIGQ9Ik00LjU2Ni01LjAxM0wtLjY1Mi0uMjYzIi8+PHBhdGggZD0iTTEuNTQ0LTUuMDEzbC01LjIxOCA0Ljc1Ii8+PHBhdGggZD0iTS0xLjU2Ni01LjAxM2wtNS4yMTggNC43NSIvPjwvZz48L21hcmtlcj48bWFya2VyIG9yaWVudD0iYXV0byIgb3ZlcmZsb3c9InZpc2libGUiPjxwYXRoIGQ9Ik0tNS42LTUuNjU3TC0xMS4yNTcgMC01LjYgNS42NTcuMDU3IDAtNS42LTUuNjU3eiIgZmlsbC1ydWxlPSJldmVub2RkIiBzdHJva2U9IiMwMDAiIHN0cm9rZS13aWR0aD0iLjhwdCIvPjwvbWFya2VyPjxtYXJrZXIgb3JpZW50PSJhdXRvIiBvdmVyZmxvdz0idmlzaWJsZSI+PHBhdGggZD0iTTQuNjE2IDBsLTYuOTIgNHYtOGw2LjkyIDR6IiBmaWxsLXJ1bGU9ImV2ZW5vZGQiIHN0cm9rZT0iIzAwMCIgc3Ryb2tlLXdpZHRoPSIuOHB0Ii8+PC9tYXJrZXI+PG1hcmtlciBvcmllbnQ9ImF1dG8iIG92ZXJmbG93PSJ2aXNpYmxlIj48cGF0aCBkPSJNLTEwLjY5LTQuNDM3TDEuMzI4LS4wMTdsLTEyLjAxOCA0LjQyYzEuOTItMi42MSAxLjkxLTYuMTggMC04Ljg0eiIgZm9udC1zaXplPSIxMiIgZmlsbC1ydWxlPSJldmVub2RkIiBzdHJva2Utd2lkdGg9Ii42ODc1IiBzdHJva2UtbGluZWpvaW49InJvdW5kIi8+PC9tYXJrZXI+PG1hcmtlciBvcmllbnQ9ImF1dG8iIG92ZXJmbG93PSJ2aXNpYmxlIj48cGF0aCBkPSJNLTQuNjE2IDBsNi45Mi00djhsLTYuOTItNHoiIGZpbGwtcnVsZT0iZXZlbm9kZCIgc3Ryb2tlPSIjMDAwIiBzdHJva2Utd2lkdGg9Ii44cHQiLz48L21hcmtlcj48bWFya2VyIG9yaWVudD0iYXV0byIgb3ZlcmZsb3c9InZpc2libGUiPjxwYXRoIGQ9Ik0xMCAwbDQtNEwwIDBsMTQgNC00LTR6IiBmaWxsLXJ1bGU9ImV2ZW5vZGQiIHN0cm9rZT0iIzAwMCIgc3Ryb2tlLXdpZHRoPSIuOHB0Ii8+PC9tYXJrZXI+PG1hcmtlciBvcmllbnQ9ImF1dG8iIG92ZXJmbG93PSJ2aXNpYmxlIj48cGF0aCBkPSJNMTAuNjkgNC40MzdMLTEuMzI4LjAxN2wxMi4wMTgtNC40MmMtMS45MiAyLjYxLTEuOTEgNi4xOCAwIDguODR6IiBmb250LXNpemU9IjEyIiBmaWxsLXJ1bGU9ImV2ZW5vZGQiIHN0cm9rZS13aWR0aD0iLjY4NzUiIHN0cm9rZS1saW5lam9pbj0icm91bmQiLz48L21hcmtlcj48L2RlZnM+PHBhdGggZD0iTTEzLjE4IDEzLjE0NnYtNS44MWwtNS44MSA1LjgxaDUuODF6IiBzdHJva2U9IiMwMDAiIHN0cm9rZS13aWR0aD0iMS42NDMiLz48cGF0aCBkPSJNMTIuNzYyIDEyLjcyN2wtNi41MS02LjUxIiBmaWxsPSJub25lIiBzdHJva2U9IiMwMDAiIHN0cm9rZS13aWR0aD0iMi40ODIiIHN0cm9rZS1saW5lY2FwPSJyb3VuZCIvPjwvc3ZnPg==);background-size:cover;position:absolute;border-radius:3px 0 0}.leaflet-oldie .leaflet-control-minimap-toggle-display{background-image:url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABIAAAASCAYAAABWzo5XAAAABHNCSVQICAgIfAhkiAAAAAlwSFlzAAAOxAAADsQBlSsOGwAAAH1JREFUOI3t0DEKAjEQheFP8ERWqfUCeg9rL+adbKxkYRFEY7ERhkUhCZb7wyuGYX5ewkIvK6z/ITpjxOHHPtWKrsh4fJEl3GpF29LoI9sHSS6pZjeTnTD0iOayjFevCI7hOKaJZHpObNIsih/b3WiDSzgaS+5lfrY0Wph4A4kFM89VzdVFAAAAAElFTkSuQmCC)}.leaflet-control-minimap-toggle-display-bottomright{bottom:0;right:0}.leaflet-control-minimap-toggle-display-topleft{top:0;left:0;-webkit-transform:rotate(180deg);transform:rotate(180deg)}.leaflet-control-minimap-toggle-display-bottomleft{bottom:0;left:0;-webkit-transform:rotate(90deg);transform:rotate(90deg)}.leaflet-control-minimap-toggle-display-topright{top:0;right:0;-webkit-transform:rotate(270deg);transform:rotate(270deg)}.leaflet-oldie .leaflet-control-minimap{border:1px solid #999}.leaflet-oldie .leaflet-control-minimap a{background-color:#fff}.leaflet-oldie .leaflet-control-minimap a.minimized{filter:progid:DXImageTransform.Microsoft.BasicImage(rotation=2)}</style>
<script>(function(factory,window){if(typeof define==="function"&&define.amd){define(["leaflet"],factory)}else if(typeof exports==="object"){module.exports=factory(require("leaflet"))}if(typeof window!=="undefined"&&window.L){window.L.Control.MiniMap=factory(L);window.L.control.minimap=function(layer,options){return new window.L.Control.MiniMap(layer,options)}}})(function(L){var MiniMap=L.Control.extend({includes:L.Mixin.Events,options:{position:"bottomright",toggleDisplay:false,zoomLevelOffset:-5,zoomLevelFixed:false,centerFixed:false,zoomAnimation:false,autoToggleDisplay:false,minimized:false,width:150,height:150,collapsedWidth:19,collapsedHeight:19,aimingRectOptions:{color:"#ff7800",weight:1,clickable:false},shadowRectOptions:{color:"#000000",weight:1,clickable:false,opacity:0,fillOpacity:0},strings:{hideText:"Hide MiniMap",showText:"Show MiniMap"},mapOptions:{}},initialize:function(layer,options){L.Util.setOptions(this,options);this.options.aimingRectOptions.clickable=false;this.options.shadowRectOptions.clickable=false;this._layer=layer},onAdd:function(map){this._mainMap=map;this._container=L.DomUtil.create("div","leaflet-control-minimap");this._container.style.width=this.options.width+"px";this._container.style.height=this.options.height+"px";L.DomEvent.disableClickPropagation(this._container);L.DomEvent.on(this._container,"mousewheel",L.DomEvent.stopPropagation);var mapOptions={attributionControl:false,dragging:!this.options.centerFixed,zoomControl:false,zoomAnimation:this.options.zoomAnimation,autoToggleDisplay:this.options.autoToggleDisplay,touchZoom:this.options.centerFixed?"center":!this._isZoomLevelFixed(),scrollWheelZoom:this.options.centerFixed?"center":!this._isZoomLevelFixed(),doubleClickZoom:this.options.centerFixed?"center":!this._isZoomLevelFixed(),boxZoom:!this._isZoomLevelFixed(),crs:map.options.crs};mapOptions=L.Util.extend(this.options.mapOptions,mapOptions);this._miniMap=new L.Map(this._container,mapOptions);this._miniMap.addLayer(this._layer);this._mainMapMoving=false;this._miniMapMoving=false;this._userToggledDisplay=false;this._minimized=false;if(this.options.toggleDisplay){this._addToggleButton()}this._miniMap.whenReady(L.Util.bind(function(){this._aimingRect=L.rectangle(this._mainMap.getBounds(),this.options.aimingRectOptions).addTo(this._miniMap);this._shadowRect=L.rectangle(this._mainMap.getBounds(),this.options.shadowRectOptions).addTo(this._miniMap);this._mainMap.on("moveend",this._onMainMapMoved,this);this._mainMap.on("move",this._onMainMapMoving,this);this._miniMap.on("movestart",this._onMiniMapMoveStarted,this);this._miniMap.on("move",this._onMiniMapMoving,this);this._miniMap.on("moveend",this._onMiniMapMoved,this)},this));return this._container},addTo:function(map){L.Control.prototype.addTo.call(this,map);var center=this.options.centerFixed||this._mainMap.getCenter();this._miniMap.setView(center,this._decideZoom(true));this._setDisplay(this.options.minimized);return this},onRemove:function(map){this._mainMap.off("moveend",this._onMainMapMoved,this);this._mainMap.off("move",this._onMainMapMoving,this);this._miniMap.off("moveend",this._onMiniMapMoved,this);this._miniMap.removeLayer(this._layer)},changeLayer:function(layer){this._miniMap.removeLayer(this._layer);this._layer=layer;this._miniMap.addLayer(this._layer)},_addToggleButton:function(){this._toggleDisplayButton=this.options.toggleDisplay?this._createButton("",this._toggleButtonInitialTitleText(),"leaflet-control-minimap-toggle-display leaflet-control-minimap-toggle-display-"+this.options.position,this._container,this._toggleDisplayButtonClicked,this):undefined;this._toggleDisplayButton.style.width=this.options.collapsedWidth+"px";this._toggleDisplayButton.style.height=this.options.collapsedHeight+"px"},_toggleButtonInitialTitleText:function(){if(this.options.minimized){return this.options.strings.showText}else{return this.options.strings.hideText}},_createButton:function(html,title,className,container,fn,context){var link=L.DomUtil.create("a",className,container);link.innerHTML=html;link.href="#";link.title=title;var stop=L.DomEvent.stopPropagation;L.DomEvent.on(link,"click",stop).on(link,"mousedown",stop).on(link,"dblclick",stop).on(link,"click",L.DomEvent.preventDefault).on(link,"click",fn,context);return link},_toggleDisplayButtonClicked:function(){this._userToggledDisplay=true;if(!this._minimized){this._minimize()}else{this._restore()}},_setDisplay:function(minimize){if(minimize!==this._minimized){if(!this._minimized){this._minimize()}else{this._restore()}}},_minimize:function(){if(this.options.toggleDisplay){this._container.style.width=this.options.collapsedWidth+"px";this._container.style.height=this.options.collapsedHeight+"px";this._toggleDisplayButton.className+=" minimized-"+this.options.position;this._toggleDisplayButton.title=this.options.strings.showText}else{this._container.style.display="none"}this._minimized=true;this._onToggle()},_restore:function(){if(this.options.toggleDisplay){this._container.style.width=this.options.width+"px";this._container.style.height=this.options.height+"px";this._toggleDisplayButton.className=this._toggleDisplayButton.className.replace("minimized-"+this.options.position,"");this._toggleDisplayButton.title=this.options.strings.hideText}else{this._container.style.display="block"}this._minimized=false;this._onToggle()},_onMainMapMoved:function(e){if(!this._miniMapMoving){var center=this.options.centerFixed||this._mainMap.getCenter();this._mainMapMoving=true;this._miniMap.setView(center,this._decideZoom(true));this._setDisplay(this._decideMinimized())}else{this._miniMapMoving=false}this._aimingRect.setBounds(this._mainMap.getBounds())},_onMainMapMoving:function(e){this._aimingRect.setBounds(this._mainMap.getBounds())},_onMiniMapMoveStarted:function(e){if(!this.options.centerFixed){var lastAimingRect=this._aimingRect.getBounds();var sw=this._miniMap.latLngToContainerPoint(lastAimingRect.getSouthWest());var ne=this._miniMap.latLngToContainerPoint(lastAimingRect.getNorthEast());this._lastAimingRectPosition={sw:sw,ne:ne}}},_onMiniMapMoving:function(e){if(!this.options.centerFixed){if(!this._mainMapMoving&&this._lastAimingRectPosition){this._shadowRect.setBounds(new L.LatLngBounds(this._miniMap.containerPointToLatLng(this._lastAimingRectPosition.sw),this._miniMap.containerPointToLatLng(this._lastAimingRectPosition.ne)));this._shadowRect.setStyle({opacity:1,fillOpacity:.3})}}},_onMiniMapMoved:function(e){if(!this._mainMapMoving){this._miniMapMoving=true;this._mainMap.setView(this._miniMap.getCenter(),this._decideZoom(false));this._shadowRect.setStyle({opacity:0,fillOpacity:0})}else{this._mainMapMoving=false}},_isZoomLevelFixed:function(){var zoomLevelFixed=this.options.zoomLevelFixed;return this._isDefined(zoomLevelFixed)&&this._isInteger(zoomLevelFixed)},_decideZoom:function(fromMaintoMini){if(!this._isZoomLevelFixed()){if(fromMaintoMini){return this._mainMap.getZoom()+this.options.zoomLevelOffset}else{var currentDiff=this._miniMap.getZoom()-this._mainMap.getZoom();var proposedZoom=this._miniMap.getZoom()-this.options.zoomLevelOffset;var toRet;if(currentDiff>this.options.zoomLevelOffset&&this._mainMap.getZoom()<this._miniMap.getMinZoom()-this.options.zoomLevelOffset){if(this._miniMap.getZoom()>this._lastMiniMapZoom){toRet=this._mainMap.getZoom()+1;this._miniMap.setZoom(this._miniMap.getZoom()-1)}else{toRet=this._mainMap.getZoom()}}else{toRet=proposedZoom}this._lastMiniMapZoom=this._miniMap.getZoom();return toRet}}else{if(fromMaintoMini){return this.options.zoomLevelFixed}else{return this._mainMap.getZoom()}}},_decideMinimized:function(){if(this._userToggledDisplay){return this._minimized}if(this.options.autoToggleDisplay){if(this._mainMap.getBounds().contains(this._miniMap.getBounds())){return true}return false}return this._minimized},_isInteger:function(value){return typeof value==="number"},_isDefined:function(value){return typeof value!=="undefined"},_onToggle:function(){L.Util.requestAnimFrame(function(){L.DomEvent.on(this._container,"transitionend",this._fireToggleEvents,this);if(!L.Browser.any3d){L.Util.requestAnimFrame(this._fireToggleEvents,this)}},this)},_fireToggleEvents:function(){L.DomEvent.off(this._container,"transitionend",this._fireToggleEvents,this);var data={minimized:this._minimized};this.fire(this._minimized?"minimize":"restore",data);this.fire("toggle",data)}});L.Map.mergeOptions({miniMapControl:false});L.Map.addInitHook(function(){if(this.options.miniMapControl){this.miniMapControl=(new MiniMap).addTo(this)}});return MiniMap},window);</script>
<script>LeafletWidget.methods.addMiniMap =
  function(tilesURL, tilesProvider, position,
  width, height, collapsedWidth, collapsedHeight , zoomLevelOffset,
  zoomLevelFixed, centerFixed, zoomAnimation , toggleDisplay, autoToggleDisplay,
  minimized, aimingRectOptions, shadowRectOptions, strings, mapOptions) {

    (function() {
      if(this.minimap) {
        this.minimap.removeFrom( this );
      }

      // determin the tiles for the minimap
      // default to OSM tiles
      layer = new L.tileLayer('http://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png');
      if(tilesProvider) {
        // use a custom tiles provider if specified.
        layer = new L.tileLayer.provider(tilesProvider);
      } else if(tilesURL) {
        // else use a custom tiles URL if specified.
        layer = new L.tileLayer(tilesURL);
      }

      this.minimap = new L.Control.MiniMap(layer, {
        position: position,
        width: width,
        height: height,
        collapsedWidth: collapsedWidth,
        collapsedHeight: collapsedWidth,
        zoomLevelOffset: zoomLevelOffset,
        zoomLevelFixed: zoomLevelFixed,
        centerFixed: centerFixed,
        zoomAnimation: zoomAnimation,
        toggleDisplay: toggleDisplay,
        autoToggleDisplay: autoToggleDisplay,
        minimized: minimized,
        aimingRectOptions: aimingRectOptions,
        shadowRectOptions: shadowRectOptions,
        strings: strings,
        mapOptions: mapOptions
      });
      this.minimap.addTo(this);
    }).call(this);
  };
</script>

</head>
<body>
<div id="htmlwidget_container">
  <div id="htmlwidget-066bb86a2665d8e67d60" style="width:100%;height:400px;" class="leaflet html-widget"></div>
</div>
<script type="application/json" data-for="htmlwidget-066bb86a2665d8e67d60">{"x":{"options":{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}}},"calls":[{"method":"addProviderTiles","args":["OpenStreetMap",null,null,{"errorTileUrl":"","noWrap":false,"detectRetina":false}]},{"method":"addMarkers","args":[[51.901879,51.894094,51.892602,51.890526,51.901108,45.761175,45.191308,47.38215,47.394043,47.425434,48.641022],[-8.469298,-8.484711,-8.505056,-8.463617,-8.481898,11.747989,5.730674,0.677217,0.681406,0.701163,1.823459],null,null,"Pizzerias",{"interactive":true,"draggable":false,"keyboard":true,"title":"","alt":"","zIndexOffset":0,"opacity":1,"riseOnHover":false,"riseOffset":250},["<b> Novecento <\/b> <br> <br> City: <b> Cork <\/b>  in <b> Ireland <\/b> <br> Open on: <b> 1st January 2012 <\/b> <\/b> <br> You can find here  <b> 20 <\/b>  pizzas <br> The special pizza is the: <b> margherita <\/b> <br> Take away: <b> yes <\/b> <br> They sell other thins? <b> no <\/b>","<b> Pizza Amore <\/b> <br> <br> City: <b> Cork <\/b>  in <b> Ireland <\/b> <br> Open on: <b> 23rd April 2012 <\/b> <\/b> <br> You can find here  <b> 12 <\/b>  pizzas <br> The special pizza is the: <b> diavola <\/b> <br> Take away: <b> yes <\/b> <br> They sell other thins? <b> yes <\/b>","<b> Heaven Pizza <\/b> <br> <br> City: <b> Cork <\/b>  in <b> Ireland <\/b> <br> Open on: <b> 12th April 2012 <\/b> <\/b> <br> You can find here  <b> 50 <\/b>  pizzas <br> The special pizza is the: <b> hawai <\/b> <br> Take away: <b> no <\/b> <br> They sell other thins? <b> yes <\/b>","<b> Zicos Pizza <\/b> <br> <br> City: <b> Cork <\/b>  in <b> Ireland <\/b> <br> Open on: <b> 12th July 2014 <\/b> <\/b> <br> You can find here  <b> 5 <\/b>  pizzas <br> The special pizza is the: <b> quattro stagioni <\/b> <br> Take away: <b> yes <\/b> <br> They sell other thins? <b> yes <\/b>","<b> Franciscan Well Brewery & Brewpub <\/b> <br> <br> City: <b> Cork <\/b>  in <b> Ireland <\/b> <br> Open on: <b> 04th January 2013 <\/b> <\/b> <br> You can find here  <b> 23 <\/b>  pizzas <br> The special pizza is the: <b> scamorza <\/b> <br> Take away: <b> no <\/b> <br> They sell other thins? <b> no <\/b>","<b> La Matta <\/b> <br> <br> City: <b> Bassano del Grappa <\/b>  in <b> Italy <\/b> <br> Open on: <b> 09th Decembre 2019 <\/b> <\/b> <br> You can find here  <b> 60 <\/b>  pizzas <br> The special pizza is the: <b> margherita <\/b> <br> Take away: <b> yes <\/b> <br> They sell other thins? <b> yes\n <\/b>","<b> Come Prima <\/b> <br> <br> City: <b> Grenoble <\/b>  in <b> France <\/b> <br> Open on: <b> 10th October 2019 <\/b> <\/b> <br> You can find here  <b> 15 <\/b>  pizzas <br> The special pizza is the: <b> reine <\/b> <br> Take away: <b> yes <\/b> <br> They sell other thins? <b> yes\n <\/b>","<b> Speed Rabbit Pizza <\/b> <br> <br> City: <b> Tours <\/b>  in <b> France <\/b> <br> Open on: <b> 9th December 2012\n <\/b> <\/b> <br> You can find here  <b> 14 <\/b>  pizzas <br> The special pizza is the: <b> internet explorer 8 <\/b> <br> Take away: <b> yes <\/b> <br> They sell other thins? <b> yes <\/b>","<b> Di Gusto <\/b> <br> <br> City: <b> Tours <\/b>  in <b> France <\/b> <br> Open on: <b> 1st June 2019\n <\/b> <\/b> <br> You can find here  <b> 11 <\/b>  pizzas <br> The special pizza is the: <b> arancini\n <\/b> <br> Take away: <b> yes <\/b> <br> They sell other thins? <b> yes <\/b>","<b> Restaurant Tablapizza Tours <\/b> <br> <br> City: <b> Tours <\/b>  in <b> France <\/b> <br> Open on: <b> 1st June 2018 <\/b> <\/b> <br> You can find here  <b> 15 <\/b>  pizzas <br> The special pizza is the: <b> charcutière <\/b> <br> Take away: <b> yes <\/b> <br> They sell other thins? <b> yes <\/b>","<b> Villabate <\/b> <br> <br> City: <b> Rambouillet <\/b>  in <b> France <\/b> <br> Open on: <b> 9th December 1980 <\/b> <\/b> <br> You can find here  <b> 31 <\/b>  pizzas <br> The special pizza is the: <b> l'huile d'olive bio <\/b> <br> Take away: <b> yes <\/b> <br> They sell other thins? <b> yes <\/b>"],null,null,null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null]},{"method":"addPolygons","args":[[[[{"lng":[4.579,4.595,4.582,4.516,4.148,4.122,4.051,3.635,3.511,3.312,3.149,3.039,3.05,3.139,3.175,2.947,2.674,2.672,2.533,2.257,2.017,1.935,1.73,1.786,1.737,1.494,1.44,1.357,1.17,1.079,0.708,0.645,0.676,0.36,0.293,0.176,-0.011,-0.313,-0.567,-0.725,-0.752,-0.946,-1.264,-1.309,-1.27,-1.332,-1.355,-1.444,-1.472,-1.414,-1.383,-1.514,-1.609,-1.624,-1.73,-1.791,-1.61,-1.448,-1.307,-1.26,-1.195,-1.004,-1.168,-1.262,-1.156,-1.091,-0.989,-1.237,-1.243,-1.138,-1.17,-1.076,-1.1,-1.065,-1.122,-1.053,-1.161,-1.242,-1.111,-1.123,-1.202,-1.206,-1.812,-1.857,-2.142,-2.155,-2.112,-1.98,-2.054,-2.247,-2.167,-2.17,-2.295,-2.546,-2.503,-2.558,-2.432,-2.5,-2.42,-2.633,-2.575,-2.574,-2.8,-2.92,-2.879,-2.854,-2.823,-2.731,-2.683,-2.71,-2.715,-2.794,-2.89,-2.978,-2.93,-3.129,-3.08,-3.146,-3.14,-3.211,-3.103,-3.154,-3.128,-3.215,-3.213,-3.359,-3.279,-3.365,-3.306,-3.282,-3.291,-3.279,-3.294,-3.287,-3.344,-3.388,-3.404,-3.356,-3.453,-3.525,-3.537,-3.681,-3.722,-3.698,-3.651,-3.639,-3.696,-3.733,-3.749,-3.742,-3.854,-3.946,-3.992,-3.977,-4.079,-4.1,-4.144,-4.113,-4.176,-4.113,-4.214,-4.159,-4.179,-4.374,-4.346,-4.423,-4.738,-4.374,-4.33,-4.273,-4.465,-4.554,-4.544,-4.631,-4.57,-4.583,-4.533,-4.537,-4.272,-4.289,-4.267,-4.118,-4.097,-4.24,-4.28,-4.183,-4.336,-4.261,-4.458,-4.27,-4.772,-4.746,-4.79,-4.793,-4.718,-4.775,-4.671,-4.508,-4.611,-4.597,-4.478,-4.568,-4.542,-4.341,-4.299,-4.187,-4.061,-4.007,-3.967,-3.976,-3.952,-3.953,-3.934,-3.921,-3.859,-3.811,-3.864,-3.82,-3.581,-3.549,-3.582,-3.504,-3.43,-3.224,-3.203,-3.237,-3.22,-3.168,-3.072,-3.123,-3.012,-3.046,-2.929,-2.946,-2.682,-2.682,-2.485,-2.318,-2.286,-2.338,-2.246,-2.213,-2.124,-2.15,-2.049,-1.972,-2.001,-1.942,-2.029,-1.937,-1.844,-1.867,-1.768,-1.347,-1.401,-1.372,-1.445,-1.571,-1.574,-1.614,-1.553,-1.561,-1.55,-1.556,-1.505,-1.56,-1.578,-1.599,-1.611,-1.567,-1.614,-1.627,-1.711,-1.68,-1.809,-1.888,-1.848,-1.947,-1.942,-1.62,-1.473,-1.267,-1.229,-1.306,-1.136,-0.962,-0.226,-0.087,0.136,0.341,0.121,0.067,0.19,0.58,1.195,1.448,1.546,1.682,1.538,1.556,1.633,1.558,1.583,1.59,1.599,1.663,1.577,1.562,1.601,1.611,1.581,1.853,2.121,2.149,2.177,2.19,2.166,2.546,2.633,2.59,2.599,2.726,2.796,2.906,2.952,3.151,3.258,3.287,3.613,3.71,3.747,4.023,4.135,4.208,4.127,4.2,4.231,4.136,4.197,4.446,4.694,4.693,4.823,4.897,4.79,4.888,4.855,4.999,5.165,5.269,5.332,5.315,5.428,5.471,5.771,5.972,5.982,6.256,6.548,6.602,6.566,6.739,6.834,6.837,6.935,7.034,7.054,7.101,7.294,7.445,7.635,7.935,8.227,8.092,7.84,7.733,7.744,7.577,7.569,7.622,7.512,7.585,7.388,7.181,7.2,6.983,6.999,6.879,7.016,7.057,6.943,6.955,6.7,6.433,6.465,6.438,6.111,6.157,6.064,6.17,6.102,6.124,5.964,5.995,5.957,6.135,6.31,6.238,6.304,6.39,6.513,6.805,6.774,6.865,6.797,6.897,6.873,6.937,7.045,6.94,6.818,6.812,7.001,7,7.186,7.111,7.138,7.08,6.969,6.894,6.768,6.627,6.674,6.746,6.751,7.007,7.066,7.077,6.963,6.933,6.854,6.945,6.893,6.887,7.008,7.361,7.685,7.7,7.501,7.53,7.487,7.158,7.139,6.963,6.855,6.747,6.716,6.584,6.697,6.646,6.386,6.364,6.201,6.151,6.096,6.115,5.921,5.881,5.947,5.828,5.809,5.687,5.341,5.374,5.313,5.052,4.969,4.893,4.861,4.877,4.824,4.872,4.829,4.917,4.855,4.613,4.558,4.579],"lat":[43.399,43.406,43.431,43.455,43.477,43.547,43.557,43.378,43.273,43.254,43.139,42.942,42.55,42.515,42.435,42.482,42.405,42.341,42.333,42.438,42.349,42.454,42.495,42.574,42.618,42.653,42.604,42.719,42.708,42.788,42.861,42.783,42.691,42.724,42.675,42.736,42.684,42.849,42.781,42.891,42.967,42.954,43.045,43.07,43.119,43.108,43.028,43.049,43.088,43.129,43.253,43.294,43.252,43.306,43.296,43.373,43.427,43.642,44.172,44.543,44.657,44.648,44.776,44.632,45.476,45.562,45.583,45.706,45.782,45.8,45.856,45.907,45.944,45.956,46.004,46.004,46.155,46.157,46.261,46.302,46.316,46.266,46.493,46.61,46.819,46.89,46.897,47.029,47.094,47.132,47.166,47.268,47.234,47.291,47.329,47.376,47.416,47.493,47.495,47.506,47.522,47.559,47.487,47.548,47.564,47.537,47.557,47.542,47.605,47.649,47.589,47.645,47.574,47.661,47.555,47.597,47.468,47.484,47.58,47.642,47.718,47.72,47.751,47.696,47.645,47.686,47.688,47.71,47.748,47.777,47.791,47.794,47.794,47.772,47.742,47.814,47.809,47.732,47.695,47.809,47.763,47.777,47.804,47.818,47.82,47.829,47.827,47.803,47.849,47.797,47.792,47.904,47.896,47.854,47.878,47.863,47.913,47.935,47.907,47.862,47.868,47.832,47.804,47.798,47.84,47.963,48.039,48.11,48.081,48.154,48.239,48.168,48.25,48.279,48.282,48.319,48.342,48.284,48.296,48.274,48.259,48.218,48.256,48.254,48.279,48.296,48.315,48.36,48.327,48.445,48.329,48.366,48.362,48.416,48.475,48.519,48.576,48.551,48.578,48.608,48.572,48.609,48.637,48.677,48.633,48.686,48.677,48.727,48.723,48.699,48.671,48.619,48.599,48.675,48.615,48.629,48.671,48.719,48.67,48.746,48.788,48.839,48.797,48.868,48.835,48.79,48.784,48.853,48.884,48.76,48.821,48.785,48.754,48.721,48.492,48.531,48.644,48.689,48.664,48.62,48.646,48.573,48.604,48.633,48.637,48.537,48.492,48.52,48.648,48.703,48.709,48.635,48.602,48.631,48.658,48.694,48.655,48.744,48.822,48.834,48.934,48.914,48.894,49.025,49.02,49.038,49.001,49.037,49.214,49.221,49.232,49.211,49.325,49.33,49.373,49.535,49.627,49.675,49.726,49.643,49.697,49.695,49.608,49.539,49.354,49.397,49.282,49.298,49.405,49.435,49.463,49.51,49.705,49.852,49.968,50.105,50.214,50.181,50.281,50.361,50.365,50.405,50.535,50.539,50.527,50.506,50.572,50.725,50.724,50.804,50.867,50.971,50.986,51.034,51.007,51.018,51.044,51.089,50.946,50.919,50.85,50.809,50.725,50.692,50.752,50.79,50.701,50.528,50.492,50.303,50.351,50.359,50.257,50.273,50.135,50.131,50.073,50.019,49.955,49.937,49.996,50.085,50.168,50.138,49.969,49.906,49.792,49.799,49.693,49.696,49.654,49.611,49.597,49.497,49.563,49.491,49.451,49.51,49.427,49.367,49.347,49.164,49.151,49.213,49.222,49.19,49.113,49.156,49.115,49.184,49.054,49.058,48.974,48.806,48.641,48.395,48.326,48.12,48.034,47.972,47.697,47.577,47.433,47.441,47.494,47.494,47.452,47.352,47.372,47.334,47.288,47.243,47.039,46.929,46.89,46.762,46.576,46.545,46.416,46.368,46.285,46.251,46.197,46.183,46.128,46.141,46.244,46.277,46.367,46.34,46.405,46.394,46.348,46.28,46.139,46.124,46.052,46.065,45.922,45.847,45.835,45.726,45.64,45.505,45.404,45.327,45.256,45.214,45.208,45.137,45.16,45.103,45.02,45.013,44.906,44.839,44.713,44.681,44.678,44.573,44.529,44.432,44.421,44.361,44.236,44.116,44.174,44.041,43.875,43.784,43.749,43.654,43.546,43.542,43.411,43.419,43.347,43.277,43.266,43.167,43.145,43.088,43.116,43.027,43.029,43.083,43.124,43.108,43.068,43.05,43.115,43.18,43.214,43.295,43.36,43.324,43.425,43.404,43.454,43.411,43.424,43.39,43.376,43.379,43.333,43.354,43.377,43.399]},{"lng":[-3.142,-3.142,-3.142,-3.142],"lat":[47.733,47.733,47.733,47.733]},{"lng":[-3.142,-3.136,-3.136,-3.142],"lat":[47.733,47.729,47.735,47.733]},{"lng":[-3.178,-3.179,-3.179,-3.178],"lat":[47.736,47.737,47.736,47.736]},{"lng":[-3.179,-3.192,-3.199,-3.192,-3.179],"lat":[47.736,47.748,47.746,47.732,47.736]},{"lng":[-3.279,-3.278,-3.278,-3.279],"lat":[47.794,47.794,47.795,47.794]},{"lng":[-3.278,-3.275,-3.277,-3.278],"lat":[47.795,47.795,47.798,47.795]},{"lng":[-3.388,-3.387,-3.388,-3.388],"lat":[47.814,47.816,47.817,47.814]},{"lng":[-3.388,-3.389,-3.393,-3.388],"lat":[47.817,47.829,47.823,47.817]},{"lng":[-3.525,-3.524,-3.529,-3.525],"lat":[47.809,47.81,47.811,47.809]},{"lng":[-3.529,-3.54,-3.534,-3.54,-3.543,-3.529],"lat":[47.811,47.828,47.84,47.836,47.822,47.811]},{"lng":[-3.534,-3.528,-3.528,-3.534],"lat":[47.84,47.844,47.845,47.84]},{"lng":[-3.528,-3.523,-3.542,-3.544,-3.528],"lat":[47.845,47.849,47.865,47.858,47.845]},{"lng":[-3.696,-3.695,-3.698,-3.696],"lat":[47.827,47.828,47.831,47.827]},{"lng":[-4.113,-4.111,-4.111,-4.113],"lat":[47.935,47.936,47.938,47.935]},{"lng":[-4.111,-4.094,-4.098,-4.116,-4.115,-4.117,-4.101,-4.111],"lat":[47.938,47.962,47.972,47.976,47.984,47.975,47.971,47.938]},{"lng":[-4.105,-4.098,-4.1,-4.105],"lat":[47.943,47.942,47.943,47.943]},{"lng":[-4.098,-4.085,-4.076,-4.083,-4.098],"lat":[47.942,47.95,47.949,47.954,47.942]},{"lng":[-4.095,-4.069,-4.095,-4.095],"lat":[47.941,47.933,47.941,47.941]},{"lng":[-4.183,-4.186,-4.189,-4.183],"lat":[47.838,47.839,47.836,47.838]},{"lng":[-4.217,-4.221,-4.217,-4.217],"lat":[47.811,47.815,47.811,47.811]},{"lng":[-4.536,-4.535,-4.535,-4.536],"lat":[48.017,48.017,48.017,48.017]},{"lng":[-4.535,-4.53,-4.488,-4.541,-4.535],"lat":[48.017,48.035,48.039,48.037,48.017]},{"lng":[-4.321,-4.32,-4.321,-4.321],"lat":[48.323,48.323,48.323,48.323]},{"lng":[-4.32,-4.318,-4.317,-4.32],"lat":[48.323,48.323,48.325,48.323]},{"lng":[-4.061,-4.057,-4.058,-4.061],"lat":[48.677,48.67,48.675,48.677]},{"lng":[-3.859,-3.859,-3.859,-3.859],"lat":[48.615,48.614,48.614,48.615]},{"lng":[-3.859,-3.862,-3.871,-3.85,-3.859],"lat":[48.614,48.608,48.604,48.604,48.614]},{"lng":[1.632,1.631,1.642,1.632],"lat":[50.361,50.36,50.352,50.361]},{"lng":[5.515,5.519,5.514,5.515],"lat":[43.204,43.209,43.203,43.204]},{"lng":[5.006,5.082,5.23,5.145,5.114,5.021,5.006],"lat":[43.47,43.4,43.465,43.459,43.523,43.556,43.47]},{"lng":[1.981,2.014,1.979,1.956,1.981],"lat":[42.448,42.452,42.495,42.458,42.448]},{"lng":[-1.298,-1.298,-1.298,-1.298],"lat":[46.294,46.294,46.294,46.294]},{"lng":[-1.288,-1.288,-1.272,-1.288],"lat":[46.289,46.289,46.29,46.289]},{"lng":[-2.766,-2.766,-2.767,-2.766],"lat":[47.539,47.539,47.544,47.539]},{"lng":[-2.716,-2.716,-2.715,-2.716],"lat":[47.583,47.583,47.582,47.583]}],[{"lng":[-2.931,-2.931,-2.928,-2.931],"lat":[47.605,47.606,47.609,47.605]}],[{"lng":[9.258,9.248,9.261,9.258],"lat":[41.339,41.348,41.342,41.339]}],[{"lng":[9.271,9.266,9.256,9.268,9.271],"lat":[41.365,41.362,41.371,41.373,41.365]}],[{"lng":[9.287,9.181,9.094,9.122,9.072,9.08,8.923,8.789,8.803,8.916,8.66,8.771,8.751,8.803,8.75,8.608,8.592,8.747,8.7,8.558,8.594,8.54,8.691,8.543,8.655,8.725,8.781,8.803,9.015,9.125,9.301,9.344,9.311,9.36,9.341,9.422,9.464,9.492,9.447,9.533,9.557,9.413,9.401,9.356,9.287,9.369,9.273,9.287],"lat":[41.484,41.366,41.394,41.441,41.444,41.477,41.489,41.558,41.641,41.687,41.739,41.811,41.845,41.894,41.933,41.895,41.964,42.05,42.113,42.146,42.168,42.237,42.276,42.368,42.416,42.584,42.557,42.603,42.641,42.732,42.678,42.738,42.835,42.923,42.994,43.012,42.986,42.805,42.668,42.545,42.142,41.955,41.694,41.617,41.607,41.597,41.53,41.484]}],[{"lng":[-2.305,-2.283,-2.399,-2.372,-2.305],"lat":[46.709,46.688,46.724,46.732,46.709]}],[{"lng":[-2.187,-2.149,-2.308,-2.226,-2.247,-2.187],"lat":[46.961,46.893,47.025,47.017,46.994,46.961]}],[{"lng":[-2.883,-2.865,-2.861,-2.885,-2.883],"lat":[47.343,47.347,47.341,47.332,47.343]}],[{"lng":[-1.177,-1.189,-1.192,-1.171,-1.177],"lat":[44.691,44.694,44.705,44.705,44.691]}],[{"lng":[-1.208,-1.232,-1.269,-1.384,-1.413,-1.238,-1.208],"lat":[45.85,45.802,45.881,45.951,46.047,45.984,45.85]}],[{"lng":[-1.113,-1.12,-1.108,-1.113],"lat":[45.953,45.961,45.959,45.953]}],[{"lng":[-1.176,-1.173,-1.154,-1.171,-1.176],"lat":[46.008,46.024,46.019,46.019,46.008]}],[{"lng":[-1.284,-1.532,-1.563,-1.49,-1.486,-1.412,-1.444,-1.293,-1.255,-1.284],"lat":[46.145,46.202,46.245,46.254,46.202,46.23,46.214,46.187,46.164,46.145]}],[{"lng":[-1.298,-1.282,-1.294,-1.298],"lat":[46.294,46.292,46.289,46.294]}],[{"lng":[5.779,5.781,5.789,5.779],"lat":[43.076,43.087,43.083,43.076]}],[{"lng":[6.244,6.205,6.163,6.211,6.244],"lat":[43.02,42.982,43,43.002,43.02]}],[{"lng":[6.435,6.473,6.511,6.439,6.435],"lat":[43.016,43.047,43.047,43.004,43.016]}],[{"lng":[6.397,6.37,6.421,6.407,6.397],"lat":[42.993,43.001,43.014,42.997,42.993]}],[{"lng":[6.361,6.358,6.364,6.367,6.361],"lat":[43.01,43.014,43.024,43.023,43.01]}],[{"lng":[5.288,5.29,5.306,5.288],"lat":[43.268,43.277,43.28,43.268]}],[{"lng":[5.312,5.292,5.318,5.312],"lat":[43.29,43.285,43.293,43.29]}],[{"lng":[7.067,7.043,7.031,7.071,7.067],"lat":[43.514,43.516,43.522,43.517,43.514]}],[{"lng":[-4.846,-4.87,-4.853,-4.846],"lat":[48.032,48.041,48.04,48.032]}],[{"lng":[-4.859,-4.867,-4.847,-4.859],"lat":[48.346,48.348,48.36,48.346]}],[{"lng":[-4.959,-4.965,-4.961,-4.953,-4.959],"lat":[48.389,48.393,48.402,48.396,48.389]}],[{"lng":[-5.073,-5.033,-5.139,-5.077,-5.073],"lat":[48.483,48.463,48.45,48.474,48.483]}],[{"lng":[4.577,4.571,4.585,4.579,4.577],"lat":[43.398,43.399,43.406,43.399,43.398]}],[{"lng":[-4.054,-4.088,-4.057,-4.054],"lat":[47.854,47.862,47.856,47.854]}],[{"lng":[-3.121,-3.056,-3.219,-3.262,-3.121],"lat":[47.323,47.308,47.294,47.371,47.323]}],[{"lng":[-2.948,-2.994,-2.939,-2.948],"lat":[47.373,47.4,47.395,47.373]}],[{"lng":[-3.422,-3.462,-3.515,-3.429,-3.422],"lat":[47.62,47.62,47.648,47.642,47.62]}],[{"lng":[-2.859,-2.856,-2.838,-2.826,-2.859],"lat":[47.56,47.591,47.607,47.588,47.56]}],[{"lng":[-2.716,-2.718,-2.711,-2.716],"lat":[47.583,47.588,47.588,47.583]}],[{"lng":[-2.81,-2.809,-2.788,-2.787,-2.81],"lat":[47.578,47.592,47.6,47.595,47.578]}],[{"lng":[-4.176,-4.177,-4.186,-4.182,-4.176],"lat":[47.851,47.846,47.86,47.861,47.851]}],[{"lng":[-3.931,-3.926,-3.929,-3.934,-3.931],"lat":[48.699,48.688,48.685,48.698,48.699]}],[{"lng":[-3.992,-4.023,-4.04,-4.006,-3.992],"lat":[48.736,48.739,48.747,48.752,48.736]}],[{"lng":[-3.584,-3.584,-3.563,-3.57,-3.584],"lat":[48.794,48.805,48.805,48.797,48.794]}],[{"lng":[-3.01,-2.998,-3,-3.017,-3.01],"lat":[48.855,48.852,48.842,48.842,48.855]}],[{"lng":[-2.991,-3.003,-3.017,-2.991],"lat":[48.867,48.856,48.86,48.867]}],[{"lng":[-3.2,-3.184,-3.185,-3.191,-3.2],"lat":[48.879,48.877,48.87,48.869,48.879]}],[{"lng":[-1.825,-1.836,-1.842,-1.825],"lat":[48.875,48.874,48.882,48.875]}],[{"lng":[-1.63,-1.629,-1.615,-1.63],"lat":[49.658,49.66,49.658,49.658]}]],[[{"lng":[-10.612,-10.605,-10.594,-10.618,-10.612],"lat":[52.05,52.049,52.045,52.049,52.05]}],[{"lng":[-10.602,-10.601,-10.614,-10.604,-10.602],"lat":[52.064,52.06,52.054,52.065,52.064]}],[{"lng":[-10.571,-10.571,-10.59,-10.571],"lat":[52.138,52.13,52.127,52.138]}],[{"lng":[-10.514,-10.513,-10.578,-10.566,-10.514],"lat":[52.111,52.097,52.074,52.088,52.111]}],[{"lng":[-10.188,-10.197,-10.196,-10.188],"lat":[53.412,53.411,53.413,53.412]}],[{"lng":[-6.122,-6.173,-6.181,-6.163,-6.122],"lat":[53.385,53.358,53.361,53.374,53.385]}],[{"lng":[-9.904,-9.891,-9.909,-9.905,-9.904],"lat":[53.394,53.383,53.384,53.419,53.394]}],[{"lng":[-6.002,-6.019,-6.039,-6.032,-6.002],"lat":[53.495,53.484,53.491,53.5,53.495]}],[{"lng":[-9.855,-9.854,-9.875,-9.848,-9.855],"lat":[53.294,53.302,53.307,53.313,53.294]}],[{"lng":[-9.98,-9.982,-9.965,-9.971,-9.98],"lat":[53.323,53.332,53.331,53.323,53.323]}],[{"lng":[-9.662,-9.686,-9.688,-9.664,-9.662],"lat":[53.32,53.318,53.324,53.326,53.32]}],[{"lng":[-9.823,-9.816,-9.805,-9.823],"lat":[53.29,53.302,53.302,53.29]}],[{"lng":[-9.641,-9.626,-9.656,-9.641],"lat":[53.325,53.312,53.311,53.325]}],[{"lng":[-7.231,-7.221,-7.22,-7.241,-7.231],"lat":[55.435,55.435,55.433,55.434,55.435]}],[{"lng":[-10.153,-10.146,-10.162,-10.161,-10.153],"lat":[53.518,53.516,53.511,53.518,53.518]}],[{"lng":[-10.147,-10.138,-10.16,-10.16,-10.147],"lat":[53.506,53.504,53.503,53.505,53.506]}],[{"lng":[-10.154,-10.173,-10.181,-10.164,-10.154],"lat":[53.533,53.526,53.538,53.542,53.533]}],[{"lng":[-9.98,-9.995,-10.004,-9.987,-9.98],"lat":[53.563,53.558,53.563,53.567,53.563]}],[{"lng":[-10.276,-10.289,-10.303,-10.291,-10.276],"lat":[53.607,53.611,53.605,53.622,53.607]}],[{"lng":[-10.138,-10.128,-10.082,-10.092,-10.138],"lat":[53.693,53.714,53.712,53.701,53.693]}],[{"lng":[-10.034,-10.027,-10.023,-10.04,-10.034],"lat":[53.724,53.722,53.716,53.724,53.724]}],[{"lng":[-10.215,-10.238,-10.261,-10.181,-10.215],"lat":[53.616,53.615,53.625,53.632,53.616]}],[{"lng":[-9.574,-9.567,-9.566,-9.574],"lat":[53.807,53.807,53.805,53.807]}],[{"lng":[-10,-9.946,-10.058,-10.05,-10],"lat":[53.82,53.809,53.791,53.805,53.82]}],[{"lng":[-9.627,-9.629,-9.636,-9.63,-9.627],"lat":[53.807,53.806,53.807,53.81,53.807]}],[{"lng":[-9.63,-9.636,-9.646,-9.637,-9.63],"lat":[53.81,53.816,53.82,53.819,53.81]}],[{"lng":[-9.654,-9.662,-9.678,-9.672,-9.654],"lat":[53.848,53.84,53.838,53.845,53.848]}],[{"lng":[-9.624,-9.607,-9.607,-9.622,-9.624],"lat":[53.854,53.854,53.852,53.852,53.854]}],[{"lng":[-9.624,-9.632,-9.629,-9.624],"lat":[53.854,53.856,53.856,53.854]}],[{"lng":[-9.651,-9.631,-9.629,-9.64,-9.651],"lat":[53.838,53.84,53.834,53.83,53.838]}],[{"lng":[-9.949,-9.946,-9.968,-9.967,-9.949],"lat":[53.872,53.86,53.867,53.875,53.872]}],[{"lng":[-9.568,-9.579,-9.587,-9.576,-9.568],"lat":[53.879,53.876,53.88,53.881,53.879]}],[{"lng":[-9.871,-9.856,-9.872,-9.877,-9.871],"lat":[53.997,53.983,53.973,53.99,53.997]}],[{"lng":[-10.18,-10.189,-10.167,-10.18],"lat":[54.074,54.078,54.079,54.074]}],[{"lng":[-9.887,-9.896,-9.926,-9.912,-9.887],"lat":[54.007,53.991,53.999,54.005,54.007]}],[{"lng":[-9.954,-9.965,-9.916,-9.964,-10.059,-10.066,-10.261,-10.192,-9.954],"lat":[54.017,53.954,53.966,53.876,53.919,53.974,53.978,54.019,54.017]}],[{"lng":[-10.22,-10.21,-10.242,-10.233,-10.22],"lat":[54.127,54.119,54.109,54.121,54.127]}],[{"lng":[-10.19,-10.21,-10.214,-10.201,-10.19],"lat":[54.14,54.13,54.148,54.151,54.14]}],[{"lng":[-9.142,-9.158,-9.197,-9.154,-9.142],"lat":[54.2,54.218,54.227,54.218,54.2]}],[{"lng":[-8.589,-8.58,-8.596,-8.6,-8.589],"lat":[54.308,54.304,54.292,54.305,54.308]}],[{"lng":[-8.656,-8.65,-8.671,-8.671,-8.656],"lat":[54.44,54.438,54.435,54.438,54.44]}],[{"lng":[-8.437,-8.444,-8.461,-8.446,-8.437],"lat":[54.959,54.953,54.96,54.966,54.959]}],[{"lng":[-8.482,-8.475,-8.459,-8.464,-8.482],"lat":[54.997,54.999,54.99,54.987,54.997]}],[{"lng":[-8.414,-8.442,-8.431,-8.412,-8.414],"lat":[55.017,55.027,55.053,55.029,55.017]}],[{"lng":[-8.449,-8.443,-8.461,-8.465,-8.449],"lat":[55.062,55.053,55.05,55.059,55.062]}],[{"lng":[-7.467,-7.5,-7.533,-7.489,-7.467],"lat":[55.051,55.05,55.074,55.086,55.051]}],[{"lng":[-8.37,-8.362,-8.354,-8.384,-8.37],"lat":[55.097,55.096,55.086,55.089,55.097]}],[{"lng":[-8.168,-8.152,-8.132,-8.168],"lat":[55.146,55.164,55.158,55.146]}],[{"lng":[-8.181,-8.172,-8.166,-8.175,-8.181],"lat":[55.178,55.181,55.177,55.171,55.178]}],[{"lng":[-8.463,-8.449,-8.469,-8.463],"lat":[54.986,54.975,54.977,54.986]}],[{"lng":[-8.503,-8.49,-8.569,-8.545,-8.503],"lat":[55,54.978,54.98,55.022,55]}],[{"lng":[-8.251,-8.195,-8.196,-8.221,-8.251],"lat":[55.277,55.266,55.26,55.257,55.277]}],[{"lng":[-9.528,-9.488,-9.466,-9.517,-9.528],"lat":[51.434,51.455,51.452,51.425,51.434]}],[{"lng":[-9.442,-9.447,-9.447,-9.442],"lat":[51.458,51.456,51.458,51.458]}],[{"lng":[-9.42,-9.396,-9.442,-9.437,-9.42],"lat":[51.481,51.475,51.458,51.48,51.481]}],[{"lng":[-9.395,-9.393,-9.4,-9.401,-9.395],"lat":[51.502,51.495,51.494,51.497,51.502]}],[{"lng":[-9.428,-9.436,-9.46,-9.451,-9.428],"lat":[51.505,51.491,51.493,51.501,51.505]}],[{"lng":[-9.506,-9.529,-9.54,-9.483,-9.506],"lat":[51.69,51.684,51.684,51.71,51.69]}],[{"lng":[-10.159,-10.179,-10.239,-10.205,-10.159],"lat":[51.612,51.601,51.587,51.609,51.612]}],[{"lng":[-9.806,-9.89,-9.924,-9.79,-9.806],"lat":[51.64,51.614,51.623,51.649,51.64]}],[{"lng":[-9.581,-9.541,-9.539,-9.576,-9.581],"lat":[51.498,51.509,51.507,51.494,51.498]}],[{"lng":[-9.39,-9.397,-9.36,-9.371,-9.39],"lat":[51.496,51.506,51.516,51.495,51.496]}],[{"lng":[-9.496,-9.508,-9.516,-9.515,-9.496],"lat":[51.517,51.508,51.51,51.513,51.517]}],[{"lng":[-9.343,-9.364,-9.372,-9.349,-9.343],"lat":[51.529,51.518,51.524,51.531,51.529]}],[{"lng":[-9.461,-9.472,-9.484,-9.485,-9.461],"lat":[51.523,51.517,51.516,51.52,51.523]}],[{"lng":[-9.945,-9.945,-9.95,-9.945],"lat":[51.801,51.799,51.799,51.801]}],[{"lng":[-9.903,-9.906,-9.911,-9.908,-9.903],"lat":[51.81,51.806,51.807,51.81,51.81]}],[{"lng":[-8.31,-8.286,-8.207,-8.32,-8.31],"lat":[51.892,51.883,51.878,51.845,51.892]}],[{"lng":[-10.35,-10.286,-10.429,-10.409,-10.35],"lat":[51.939,51.93,51.886,51.914,51.939]}],[{"lng":[-10.304,-10.312,-10.288,-10.304],"lat":[51.935,51.944,51.943,51.935]}],[{"lng":[-6.604,-6.603,-6.622,-6.621,-6.604],"lat":[52.121,52.116,52.111,52.116,52.121]}],[{"lng":[-10.398,-10.403,-10.417,-10.411,-10.398],"lat":[51.846,51.838,51.834,51.842,51.846]}],[{"lng":[-10.247,-10.238,-10.265,-10.261,-10.247],"lat":[51.74,51.736,51.73,51.736,51.74]}],[{"lng":[-10.217,-10.215,-10.228,-10.224,-10.217],"lat":[51.745,51.738,51.736,51.744,51.745]}],[{"lng":[-9.878,-9.892,-9.889,-9.864,-9.878],"lat":[52.284,52.3,52.304,52.296,52.284]}],[{"lng":[-9.496,-9.489,-9.511,-9.511,-9.496],"lat":[52.582,52.572,52.579,52.582,52.582]}],[{"lng":[-9.513,-9.514,-9.523,-9.526,-9.513],"lat":[52.621,52.615,52.608,52.618,52.621]}],[{"lng":[-9.035,-9.043,-9.056,-9.048,-9.035],"lat":[52.683,52.673,52.669,52.681,52.683]}],[{"lng":[-9.04,-9.042,-9.058,-9.05,-9.04],"lat":[52.693,52.685,52.68,52.689,52.693]}],[{"lng":[-9.059,-9.047,-9.074,-9.081,-9.059],"lat":[52.639,52.628,52.613,52.636,52.639]}],[{"lng":[-9.105,-9.103,-9.122,-9.105],"lat":[52.631,52.619,52.626,52.631]}],[{"lng":[-8.97,-8.96,-8.96,-8.976,-8.97],"lat":[52.712,52.712,52.71,52.707,52.712]}],[{"lng":[-8.956,-8.943,-8.934,-8.949,-8.956],"lat":[52.709,52.717,52.718,52.705,52.709]}],[{"lng":[-8.989,-8.993,-9.018,-9.011,-8.989],"lat":[52.718,52.713,52.708,52.717,52.718]}],[{"lng":[-9.019,-9.014,-9.054,-9.033,-9.019],"lat":[52.722,52.721,52.711,52.722,52.722]}],[{"lng":[-9.512,-9.532,-9.535,-9.51,-9.512],"lat":[52.81,52.811,52.816,52.818,52.81]}],[{"lng":[-9.502,-9.527,-9.557,-9.533,-9.502],"lat":[53.059,53.048,53.052,53.07,53.059]}],[{"lng":[-9.569,-9.579,-9.618,-9.592,-9.569],"lat":[53.086,53.074,53.07,53.102,53.086]}],[{"lng":[-9.071,-9.085,-9.056,-9.071],"lat":[53.159,53.166,53.164,53.159]}],[{"lng":[-8.988,-8.981,-8.966,-8.977,-8.988],"lat":[53.212,53.212,53.208,53.207,53.212]}],[{"lng":[-8.963,-8.969,-9.006,-8.985,-8.963],"lat":[53.243,53.239,53.236,53.241,53.243]}],[{"lng":[-9.742,-9.721,-9.721,-9.761,-9.742],"lat":[53.249,53.248,53.233,53.232,53.249]}],[{"lng":[-9.683,-9.724,-9.654,-9.647,-9.683],"lat":[53.229,53.266,53.284,53.232,53.229]}],[{"lng":[-9.74,-9.75,-9.752,-9.737,-9.74],"lat":[53.256,53.252,53.262,53.259,53.256]}],[{"lng":[-9.008,-9.019,-9.045,-9.022,-9.008],"lat":[53.222,53.215,53.222,53.228,53.222]}],[{"lng":[-9.677,-9.631,-9.831,-9.655,-9.677],"lat":[53.119,53.109,53.146,53.132,53.119]}],[{"lng":[-9.115,-9.116,-9.107,-9.108,-9.115],"lat":[53.143,53.149,53.15,53.147,53.143]}],[{"lng":[-9.72,-9.72,-9.704,-9.72],"lat":[53.276,53.286,53.277,53.276]}],[{"lng":[-9.658,-9.633,-9.691,-9.717,-9.658],"lat":[53.304,53.3,53.283,53.29,53.304]}],[{"lng":[-8,-7.943,-7.947,-7.996,-8.009,-7.906,-7.918,-7.855,-7.908,-7.856,-7.827,-7.871,-7.79,-7.817,-7.808,-7.708,-7.695,-7.668,-7.704,-7.715,-7.8,-7.63,-7.623,-7.519,-7.642,-7.561,-7.641,-7.686,-7.445,-7.463,-7.554,-7.52,-7.363,-7.252,-7.389,-6.923,-7.393,-7.544,-7.921,-7.856,-7.701,-8.182,-7.878,-7.867,-7.688,-7.551,-7.416,-7.327,-7.285,-7.292,-7.253,-7.15,-7.192,-7.032,-6.884,-6.885,-6.808,-6.645,-6.673,-6.634,-6.301,-6.111,-6.168,-6.316,-6.404,-6.342,-6.373,-6.243,-6.264,-6.221,-6.26,-6.338,-6.262,-6.22,-6.086,-6.083,-6.174,-6.104,-6.138,-6.21,-6.057,-6.123,-6.237,-6.095,-6.053,-6.002,-6.144,-6.145,-6.232,-6.198,-6.353,-6.357,-6.469,-6.514,-6.433,-6.385,-6.38,-6.318,-6.366,-6.487,-6.591,-6.701,-6.629,-6.649,-6.802,-6.752,-6.759,-6.929,-6.901,-6.995,-6.949,-6.989,-7.364,-7.554,-7.584,-7.64,-7.646,-7.538,-7.589,-7.821,-7.834,-7.886,-7.848,-7.932,-7.872,-8.069,-8.26,-8.177,-8.214,-8.176,-8.18,-8.436,-8.34,-8.351,-8.296,-8.391,-8.277,-8.349,-8.301,-8.314,-8.44,-8.475,-8.479,-8.507,-8.596,-8.602,-8.637,-8.597,-8.508,-8.533,-8.676,-8.764,-8.685,-8.716,-8.882,-8.852,-8.954,-9.027,-9.086,-9.149,-9.111,-9.176,-9.166,-9.174,-9.232,-9.307,-9.384,-9.388,-9.361,-9.358,-9.325,-9.323,-9.358,-9.415,-9.396,-9.457,-9.461,-9.655,-9.819,-9.839,-9.543,-9.858,-9.451,-9.457,-9.543,-9.651,-9.729,-10.166,-10.063,-10.099,-9.976,-9.954,-10.011,-9.981,-9.801,-9.833,-9.739,-9.649,-9.561,-9.827,-9.875,-9.909,-10.133,-10.225,-10.174,-10.207,-10.342,-10.33,-10.403,-10.254,-10.278,-10.184,-10.32,-9.97,-9.976,-9.924,-9.902,-9.787,-9.801,-9.752,-9.976,-9.942,-9.993,-10.196,-10.283,-10.263,-10.45,-10.471,-10.406,-10.373,-10.361,-10.177,-10.181,-10.084,-10.02,-10.024,-9.958,-9.723,-9.879,-9.83,-9.835,-9.951,-9.634,-9.689,-9.623,-9.479,-9.454,-9.053,-8.953,-8.692,-8.626,-8.621,-8.693,-8.959,-8.928,-8.973,-8.925,-8.958,-9.164,-9.318,-9.267,-9.418,-9.585,-9.549,-9.699,-9.941,-9.653,-9.61,-9.498,-9.436,-9.348,-9.476,-9.282,-9.151,-8.967,-8.929,-8.951,-8.882,-8.948,-9.01,-8.943,-8.986,-8.943,-9.524,-9.557,-9.54,-9.557,-9.616,-9.623,-9.548,-9.598,-9.556,-9.616,-9.628,-9.668,-9.628,-9.64,-9.694,-9.775,-9.909,-9.79,-9.883,-9.839,-9.901,-9.96,-10.069,-10.188,-10.037,-10.09,-10.079,-10.051,-10.022,-10.193,-9.959,-10.053,-9.85,-9.883,-9.788,-9.694,-9.914,-9.907,-9.547,-9.63,-9.558,-9.555,-9.59,-9.945,-9.909,-9.804,-9.817,-9.796,-9.8,-9.853,-9.829,-9.912,-9.826,-9.975,-9.993,-9.9,-9.988,-9.921,-9.958,-9.998,-10.096,-10.068,-10.128,-10.099,-10.059,-10.112,-10.003,-9.937,-9.894,-9.952,-9.987,-9.954,-9.852,-9.884,-9.847,-9.749,-9.864,-9.789,-9.262,-9.203,-9.263,-9.195,-9.223,-9.157,-9.138,-9.05,-8.662,-8.59,-8.508,-8.627,-8.48,-8.577,-8.506,-8.649,-8.677,-8.5,-8.471,-8.203,-8.272,-8.171,-8.111,-8.147,-8.119,-8.301,-8.287,-8.464,-8.392,-8.681,-8.804,-8.667,-8.417,-8.561,-8.341,-8.277,-8.385,-8.309,-8.458,-8.364,-8.454,-8.402,-8.334,-8.342,-8.323,-8.288,-8.139,-8],"lat":[55.218,55.208,55.2,55.186,55.174,55.193,55.159,55.168,55.137,55.131,55.177,55.226,55.258,55.201,55.181,55.165,55.102,55.159,55.227,55.174,55.219,55.279,55.203,55.125,55.047,55.035,54.999,54.952,55.052,55.14,55.201,55.288,55.322,55.283,55.384,55.237,55.023,54.744,54.699,54.63,54.609,54.466,54.288,54.219,54.207,54.125,54.155,54.121,54.168,54.122,54.203,54.226,54.336,54.419,54.348,54.279,54.212,54.168,54.073,54.042,54.112,54.007,53.979,54.016,54.015,54,53.905,53.863,53.809,53.796,53.731,53.719,53.729,53.638,53.557,53.521,53.503,53.494,53.455,53.468,53.365,53.391,53.362,53.273,52.999,52.967,52.802,52.736,52.646,52.563,52.413,52.347,52.378,52.352,52.282,52.314,52.262,52.238,52.172,52.211,52.171,52.216,52.203,52.222,52.212,52.248,52.264,52.124,52.201,52.28,52.179,52.145,52.14,52.078,52.107,52.105,52.062,52.054,51.992,51.94,51.972,52.002,51.944,51.903,51.883,51.811,51.795,51.863,51.867,51.884,51.909,51.905,51.877,51.842,51.81,51.817,51.8,51.768,51.77,51.735,51.689,51.715,51.678,51.708,51.702,51.721,51.741,51.698,51.703,51.61,51.668,51.645,51.64,51.573,51.627,51.606,51.534,51.585,51.549,51.583,51.542,51.519,51.538,51.554,51.483,51.51,51.473,51.484,51.491,51.516,51.537,51.55,51.541,51.499,51.548,51.566,51.529,51.525,51.456,51.487,51.615,51.546,51.695,51.736,51.754,51.677,51.696,51.587,51.634,51.677,51.691,51.718,51.718,51.738,51.764,51.786,51.838,51.847,51.884,51.83,51.798,51.838,51.741,51.782,51.808,51.848,51.787,51.84,51.883,51.907,51.934,51.964,51.958,52.092,52.06,52.068,52.14,52.113,52.146,52.155,52.144,52.108,52.146,52.108,52.14,52.116,52.097,52.183,52.21,52.174,52.237,52.293,52.231,52.251,52.317,52.267,52.234,52.259,52.277,52.313,52.384,52.412,52.47,52.487,52.583,52.549,52.584,52.601,52.663,52.654,52.666,52.694,52.657,52.686,52.716,52.762,52.793,52.799,52.62,52.591,52.644,52.607,52.67,52.644,52.585,52.561,52.682,52.746,52.754,52.884,52.931,52.944,53.146,53.12,53.179,53.142,53.195,53.23,53.212,53.216,53.24,53.25,53.275,53.224,53.24,53.281,53.293,53.234,53.329,53.345,53.361,53.387,53.382,53.333,53.342,53.349,53.392,53.376,53.296,53.328,53.396,53.396,53.432,53.428,53.371,53.425,53.412,53.463,53.491,53.491,53.478,53.473,53.559,53.554,53.612,53.612,53.632,53.596,53.6,53.654,53.764,53.803,53.824,53.863,53.892,53.899,53.872,53.959,53.953,53.933,53.916,53.976,53.97,53.994,54.049,54.114,54.067,54.108,54.123,54.185,54.201,54.201,54.233,54.164,54.101,54.098,54.203,54.225,54.247,54.313,54.256,54.273,54.225,54.255,54.217,54.229,54.25,54.284,54.258,54.31,54.35,54.315,54.279,54.232,54.251,54.222,54.204,54.143,54.299,54.276,54.214,54.223,54.261,54.278,54.313,54.327,54.326,54.367,54.421,54.473,54.505,54.529,54.627,54.616,54.634,54.654,54.604,54.651,54.57,54.637,54.623,54.704,54.773,54.778,54.836,54.837,54.862,54.856,54.889,54.916,54.954,55.006,55.004,55.068,55.038,55.025,55.161,55.129,55.218]}]],[[{"lng":[12.533,12.628,12.633,12.518,12.533],"lat":[35.528,35.52,35.494,35.52,35.528]}],[{"lng":[12.875,12.881,12.848,12.861,12.875],"lat":[35.875,35.855,35.868,35.876,35.875]}],[{"lng":[11.955,12.052,12.05,11.957,11.955],"lat":[36.839,36.8,36.751,36.768,36.839]}],[{"lng":[15.098,15.099,15.1,15.098],"lat":[37.5,37.5,37.488,37.5]}],[{"lng":[13.578,13.579,13.578,13.578],"lat":[37.262,37.258,37.262,37.262]}],[{"lng":[13.532,13.533,13.528,13.532],"lat":[37.285,37.284,37.28,37.285]}],[{"lng":[15.152,15.152,15.144,15.146,15.152],"lat":[36.689,36.685,36.681,36.688,36.689]}],[{"lng":[15.21,15.211,15.21,15.21],"lat":[37.728,37.734,37.728,37.728]}],[{"lng":[12.455,12.466,12.457,12.455],"lat":[37.887,37.879,37.882,37.887]}],[{"lng":[12.471,12.472,12.465,12.465,12.471],"lat":[37.87,37.866,37.864,37.869,37.87]}],[{"lng":[12.455,12.445,12.447,12.429,12.455],"lat":[37.905,37.892,37.844,37.892,37.905]}],[{"lng":[10.08,10.103,10.05,10.079,10.08],"lat":[42.619,42.576,42.575,42.597,42.619]}],[{"lng":[10.42,10.386,10.422,10.345,10.159,10.102,10.14,10.245,10.263,10.357,10.412,10.42],"lat":[42.773,42.759,42.708,42.764,42.726,42.767,42.809,42.788,42.83,42.799,42.873,42.773]}],[{"lng":[15.516,15.524,15.508,15.51,15.516],"lat":[42.14,42.139,42.129,42.138,42.14]}],[{"lng":[14.709,14.708,14.712,14.709],"lat":[42.175,42.175,42.18,42.175]}],[{"lng":[11.11,11.117,11.11,11.091,11.11],"lat":[42.264,42.252,42.238,42.253,42.264]}],[{"lng":[10.314,10.332,10.294,10.294,10.314],"lat":[42.351,42.323,42.317,42.347,42.351]}],[{"lng":[10.886,10.928,10.92,10.866,10.886],"lat":[42.389,42.356,42.317,42.353,42.389]}],[{"lng":[9.399,9.401,9.396,9.399],"lat":[41.299,41.293,41.295,41.299]}],[{"lng":[9.342,9.353,9.359,9.337,9.342],"lat":[41.309,41.303,41.294,41.296,41.309]}],[{"lng":[9.376,9.38,9.37,9.376],"lat":[41.313,41.308,41.309,41.313]}],[{"lng":[11.789,11.787,11.772,11.789],"lat":[42.09,42.089,42.1,42.09]}],[{"lng":[15.493,15.491,15.477,15.475,15.493],"lat":[42.123,42.109,42.106,42.113,42.123]}],[{"lng":[15.512,15.502,15.517,15.512],"lat":[42.122,42.12,42.128,42.122]}],[{"lng":[9.826,9.848,9.81,9.791,9.826],"lat":[43.073,43.045,43.002,43.023,43.073]}],[{"lng":[10.444,10.444,10.423,10.444],"lat":[43.355,43.354,43.356,43.355]}],[{"lng":[9.901,9.908,9.89,9.892,9.901],"lat":[43.44,43.425,43.423,43.436,43.44]}],[{"lng":[9.855,9.847,9.833,9.855],"lat":[44.051,44.033,44.047,44.051]}],[{"lng":[8.876,8.917,8.94,8.876],"lat":[44.404,44.397,44.389,44.404]}],[{"lng":[8.974,8.976,8.958,8.967,8.974],"lat":[45.984,45.962,45.965,45.984,45.984]}],[{"lng":[9.347,9.36,9.356,9.335,9.347],"lat":[41.289,41.286,41.277,41.282,41.289]}],[{"lng":[9.379,9.384,9.365,9.361,9.379],"lat":[41.308,41.298,41.288,41.297,41.308]}],[{"lng":[14.764,14.492,14.376,14.243,14.069,13.94,13.75,13.556,13.272,13.169,13.02,12.943,12.662,12.615,12.499,12.424,12.486,12.458,12.522,12.492,12.711,12.73,12.902,13.078,13.052,13.106,13.191,13.316,13.384,13.538,13.566,13.744,14.022,14.309,14.63,14.748,14.915,15.101,15.212,15.225,15.289,15.541,15.65,15.573,15.494,15.214,15.177,15.112,15.086,15.099,15.258,15.184,15.185,15.237,15.296,15.276,15.336,15.208,15.111,15.132,15.078,14.895,14.764],"lat":[36.711,36.787,36.965,37.066,37.11,37.085,37.149,37.285,37.389,37.492,37.495,37.572,37.566,37.636,37.676,37.802,37.874,37.908,38.013,38.017,38.109,38.189,38.025,38.088,38.139,38.191,38.169,38.225,38.109,38.113,38.039,37.97,38.042,38.008,38.07,38.166,38.192,38.122,38.186,38.27,38.206,38.301,38.27,38.234,38.075,37.756,37.576,37.531,37.484,37.318,37.244,37.237,37.195,37.115,37.107,37.039,37.002,36.963,36.848,36.667,36.644,36.729,36.711]},{"lng":[12.584,12.586,12.587,12.584],"lat":[37.652,37.653,37.658,37.652]},{"lng":[13.37,13.37,13.367,13.37],"lat":[38.123,38.123,38.119,38.123]}],[{"lng":[17.999,18.081,18.423,18.517,18.407,18.377,18.048,17.972,18.01,17.86,17.508,17.203,17.248,17.123,16.931,16.606,16.627,16.487,16.523,16.771,17.157,17.109,17.148,17.108,17.205,17.094,16.886,16.606,16.535,16.571,16.289,16.164,16.063,15.733,15.636,15.63,15.809,15.907,15.93,15.836,15.986,16.182,16.22,16.099,16.037,15.814,15.78,15.63,15.514,15.419,15.271,15.271,15.132,14.909,14.999,14.789,14.325,14.339,14.479,14.461,14.274,14.188,14.105,14.089,14.039,14.004,13.731,13.621,13.57,13.284,13.046,12.933,12.622,12.436,12.222,12.141,11.916,11.835,11.695,11.632,11.273,11.186,11.096,11.104,11.179,11.182,10.952,10.73,10.773,10.645,10.495,10.524,10.425,10.294,10.32,10.305,10.305,10.29,10.238,10.064,9.852,9.811,9.832,9.234,9.218,9.14,8.924,8.744,8.451,8.425,8.269,8.27,8.23,8.226,8.163,8.174,8.073,7.673,7.53,7.498,7.714,7.685,7.357,7.007,6.887,6.893,6.948,6.855,6.969,6.963,7.073,7.006,6.751,6.746,6.674,6.627,6.769,6.894,6.966,7.083,7.137,7.109,7.186,7,7.001,6.828,6.804,6.99,7.033,7.102,7.19,7.55,7.864,7.909,8.013,8.034,8.153,8.087,8.312,8.306,8.438,8.471,8.443,8.612,8.852,8.786,8.894,8.946,8.912,9.031,9.089,8.989,9.028,9.009,9.248,9.3,9.247,9.282,9.362,9.41,9.462,9.462,9.549,9.71,9.737,9.953,10.044,10.132,10.177,10.105,10.167,10.041,10.044,10.102,10.239,10.296,10.472,10.492,10.382,10.472,10.668,10.763,10.73,11.022,11.165,11.627,11.747,12.187,12.24,12.121,12.144,12.274,12.284,12.357,12.443,13.24,13.373,13.714,13.689,13.598,13.376,13.423,13.663,13.476,13.541,13.643,13.573,13.595,13.796,13.918,13.84,13.723,13.734,13.785,13.814,13.533,13.553,13.434,13.184,13.098,12.908,12.387,12.297,12.328,12.552,12.393,12.309,12.317,12.259,12.241,12.383,12.69,12.907,13.286,13.454,13.507,13.626,14.063,14.51,14.623,14.701,14.714,14.712,14.717,14.723,14.718,14.739,15.165,16.093,16.186,16.193,15.903,15.962,16.582,17.101,17.479,17.997,17.929,17.999],"lat":[40.649,40.525,40.295,40.136,39.975,39.8,39.929,40.055,40.109,40.284,40.295,40.408,40.46,40.518,40.458,40.081,39.957,39.766,39.659,39.621,39.401,39.308,39.206,39.107,39.021,38.893,38.927,38.812,38.711,38.427,38.26,38.134,37.925,37.925,38.008,38.219,38.293,38.468,38.547,38.646,38.723,38.748,38.906,39.031,39.348,39.678,39.891,40.072,40.069,39.991,40.022,40.074,40.171,40.243,40.39,40.665,40.569,40.628,40.696,40.742,40.845,40.792,40.832,40.778,40.792,40.946,41.243,41.261,41.204,41.298,41.227,41.374,41.444,41.642,41.744,41.916,42.039,42.029,42.236,42.296,42.419,42.363,42.393,42.447,42.453,42.534,42.737,42.802,42.909,42.953,42.932,43.246,43.4,43.542,43.584,43.583,43.567,43.558,43.873,44.028,44.11,44.102,44.048,44.348,44.299,44.361,44.414,44.427,44.288,44.195,44.139,44.132,44.089,44.044,43.993,43.955,43.891,43.776,43.784,43.872,44.062,44.174,44.116,44.236,44.363,44.42,44.43,44.532,44.62,44.678,44.696,44.84,44.906,45.014,45.02,45.104,45.16,45.137,45.208,45.216,45.256,45.328,45.403,45.505,45.64,45.704,45.816,45.868,45.921,45.859,45.859,45.986,45.916,45.997,46.012,46.1,46.146,46.267,46.377,46.427,46.464,46.396,46.25,46.121,46.076,45.989,45.958,45.867,45.83,45.821,45.901,45.971,45.994,46.037,46.234,46.327,46.436,46.497,46.51,46.467,46.509,46.376,46.302,46.292,46.351,46.379,46.23,46.225,46.258,46.338,46.408,46.444,46.539,46.611,46.635,46.55,46.543,46.615,46.684,46.856,46.876,46.823,46.788,46.765,46.966,47.013,46.969,47.092,47.068,47.007,46.914,46.884,46.783,46.775,46.688,46.552,46.579,46.523,46.44,46.439,46.298,46.206,46.18,46.006,45.965,45.983,45.848,45.808,45.743,45.631,45.583,45.595,45.61,45.598,45.615,45.805,45.725,45.675,45.712,45.636,45.614,45.421,45.24,45.09,44.965,44.787,44.801,44.843,44.816,44.686,44.226,43.986,43.926,43.679,43.611,43.631,43.55,42.618,42.252,42.198,42.174,42.174,42.178,42.172,42.126,42.106,42.088,41.923,41.938,41.883,41.783,41.618,41.458,41.208,41.062,40.829,40.663,40.639,40.649]},{"lng":[13.736,13.736,13.736,13.736],"lat":[43.311,43.311,43.316,43.311]},{"lng":[8.497,8.497,8.504,8.497],"lat":[44.311,44.31,44.314,44.311]},{"lng":[8.928,8.922,8.928,8.928],"lat":[44.401,44.402,44.401,44.401]},{"lng":[8.809,8.795,8.77,8.795,8.809],"lat":[44.422,44.412,44.416,44.412,44.422]},{"lng":[11.209,11.205,11.211,11.209],"lat":[42.402,42.401,42.402,42.402]},{"lng":[8.27,8.27,8.263,8.27],"lat":[44.135,44.135,44.132,44.135]},{"lng":[16.423,16.42,16.423,16.423],"lat":[41.283,41.278,41.283,41.283]},{"lng":[12.656,12.661,12.649,12.656],"lat":[41.734,41.736,41.749,41.734]},{"lng":[12.451,12.458,12.458,12.446,12.451],"lat":[41.901,41.901,41.906,41.902,41.901]},{"lng":[12.453,12.405,12.408,12.487,12.514,12.453],"lat":[43.969,43.956,43.903,43.896,43.991,43.969]},{"lng":[13.704,13.722,13.704,13.704],"lat":[37.99,37.987,37.99,37.99]},{"lng":[17.978,17.983,17.976,17.978],"lat":[40.059,40.062,40.059,40.059]},{"lng":[17.884,17.881,17.888,17.884],"lat":[40.26,40.262,40.255,40.26]},{"lng":[8.405,8.405,8.399,8.405],"lat":[40.84,40.84,40.845,40.84]},{"lng":[9.077,9.077,9.094,9.077],"lat":[39.199,39.199,39.192,39.199]},{"lng":[9.092,9.097,9.092,9.092],"lat":[39.209,39.195,39.209,39.209]},{"lng":[8.438,8.438,8.438,8.438],"lat":[39.868,39.868,39.868,39.868]},{"lng":[8.372,8.375,8.372,8.372],"lat":[39.113,39.114,39.113,39.113]},{"lng":[8.408,8.408,8.408,8.408],"lat":[39.163,39.164,39.163,39.163]},{"lng":[16.505,16.51,16.505,16.505],"lat":[41.247,41.246,41.247,41.247]}],[{"lng":[12.308,12.367,12.366,12.286,12.308],"lat":[37.956,37.925,37.906,37.915,37.956]}],[{"lng":[12.063,12.065,12.092,12.042,12.063],"lat":[37.993,37.977,37.947,37.958,37.993]}],[{"lng":[14.96,14.96,15.003,14.937,14.96],"lat":[38.432,38.414,38.37,38.402,38.432]}],[{"lng":[14.96,14.981,14.954,14.898,14.96],"lat":[38.523,38.483,38.439,38.486,38.523]}],[{"lng":[14.362,14.36,14.343,14.34,14.362],"lat":[38.554,38.533,38.533,38.548,38.554]}],[{"lng":[15.07,15.076,15.056,15.059,15.07],"lat":[38.647,38.632,38.626,38.643,38.647]}],[{"lng":[14.847,14.796,14.806,14.872,14.847],"lat":[38.538,38.561,38.583,38.581,38.538]}],[{"lng":[14.553,14.577,14.593,14.556,14.553],"lat":[38.586,38.58,38.556,38.558,38.586]}],[{"lng":[15.115,15.118,15.109,15.115],"lat":[38.666,38.661,38.664,38.666]}],[{"lng":[13.18,13.2,13.153,13.159,13.18],"lat":[38.721,38.71,38.694,38.712,38.721]}],[{"lng":[15.221,15.227,15.195,15.189,15.221],"lat":[38.812,38.777,38.78,38.796,38.812]}],[{"lng":[12.334,12.352,12.351,12.323,12.334],"lat":[38.021,37.995,37.988,37.995,38.021]}],[{"lng":[12.496,12.497,12.489,12.496],"lat":[38.012,38.009,38.012,38.012]}],[{"lng":[17.953,17.952,17.945,17.949,17.953],"lat":[40.051,40.046,40.046,40.053,40.051]}],[{"lng":[15.774,15.78,15.769,15.774],"lat":[39.876,39.872,39.872,39.876]}],[{"lng":[8.312,8.312,8.297,8.3,8.312],"lat":[39.997,39.99,39.982,39.994,39.997]}],[{"lng":[17.161,17.167,17.141,17.161,17.161],"lat":[40.453,40.448,40.451,40.458,40.453]}],[{"lng":[14.217,14.26,14.266,14.197,14.217],"lat":[40.561,40.561,40.556,40.536,40.561]}],[{"lng":[8.141,8.145,8.138,8.141],"lat":[40.605,40.605,40.602,40.605]}],[{"lng":[13.454,13.459,13.451,13.454],"lat":[40.793,40.789,40.788,40.793]}],[{"lng":[13.431,13.429,13.408,13.431],"lat":[40.804,40.791,40.785,40.804]}],[{"lng":[13.897,13.851,13.863,13.967,13.897],"lat":[40.7,40.711,40.758,40.732,40.7]}],[{"lng":[13.999,14.009,14.037,14.017,13.999],"lat":[40.748,40.769,40.765,40.739,40.748]}],[{"lng":[8.321,8.325,8.317,8.321],"lat":[39.194,39.19,39.19,39.194]}],[{"lng":[8.438,8.435,8.432,8.438],"lat":[39.868,39.858,39.862,39.868]}],[{"lng":[9.53,9.534,9.536,9.527,9.53],"lat":[39.09,39.089,39.083,39.085,39.09]}],[{"lng":[8.309,8.306,8.248,8.224,8.309],"lat":[39.188,39.1,39.108,39.16,39.188]}],[{"lng":[8.408,8.419,8.413,8.408],"lat":[39.163,39.159,39.151,39.163]}],[{"lng":[8.466,8.41,8.351,8.426,8.466,8.488,8.368,8.437,8.373,8.413,8.379,8.471,8.444,8.502,8.57,8.396,8.414,8.376,8.493,8.456,8.483,8.383,8.374,8.294,8.203,8.195,8.15,8.205,8.131,8.224,8.177,8.238,8.315,8.529,8.791,9.013,9.152,9.144,9.363,9.37,9.423,9.442,9.526,9.572,9.505,9.561,9.555,9.595,9.665,9.611,9.575,9.504,9.644,9.616,9.729,9.671,9.828,9.767,9.624,9.654,9.736,9.687,9.716,9.597,9.634,9.572,9.567,9.524,9.297,9.207,9.159,9.095,9.033,9.009,9.046,9.021,8.856,8.723,8.64,8.568,8.466],"lat":[39.057,38.957,39.098,39.109,39.057,39.085,39.214,39.292,39.373,39.427,39.456,39.599,39.758,39.713,39.865,39.909,40.017,40.036,40.087,40.177,40.285,40.339,40.491,40.593,40.568,40.618,40.579,40.685,40.733,40.886,40.939,40.953,40.844,40.824,40.923,41.125,41.155,41.247,41.208,41.183,41.178,41.102,41.157,41.1,41.008,41.038,41.004,41.023,40.995,41.003,40.928,40.913,40.92,40.89,40.844,40.785,40.528,40.392,40.251,40.143,40.077,39.984,39.928,39.346,39.299,39.245,39.147,39.095,39.213,39.226,39.181,39.214,39.166,39.123,39.056,38.984,38.877,38.938,38.865,39.043,39.057]},{"lng":[8.233,8.233,8.233,8.233],"lat":[40.94,40.936,40.94,40.94]}],[{"lng":[9.73,9.736,9.71,9.716,9.73],"lat":[40.877,40.859,40.869,40.876,40.877]}],[{"lng":[9.65,9.655,9.648,9.65],"lat":[40.89,40.888,40.887,40.89]}],[{"lng":[13.057,13.065,13.048,13.057],"lat":[40.975,40.967,40.968,40.975]}],[{"lng":[12.985,12.996,12.952,12.949,12.985],"lat":[40.935,40.932,40.877,40.919,40.935]}],[{"lng":[9.735,9.742,9.673,9.737,9.735],"lat":[40.924,40.911,40.889,40.928,40.924]}],[{"lng":[12.854,12.863,12.852,12.857,12.854],"lat":[40.95,40.93,40.925,40.941,40.95]}],[{"lng":[8.215,8.226,8.226,8.214,8.215],"lat":[40.983,40.976,40.969,40.969,40.983]}],[{"lng":[9.644,9.647,9.643,9.64,9.644],"lat":[40.983,40.98,40.978,40.983,40.983]}],[{"lng":[8.319,8.332,8.28,8.235,8.256,8.21,8.22,8.319],"lat":[41.121,41.05,41.066,41.024,40.984,40.988,41.041,41.121]}],[{"lng":[9.575,9.58,9.57,9.575],"lat":[41.07,41.063,41.062,41.07]}],[{"lng":[9.518,9.528,9.521,9.518],"lat":[41.167,41.163,41.162,41.167]}],[{"lng":[9.415,9.422,9.397,9.398,9.415],"lat":[41.208,41.19,41.19,41.198,41.208]}],[{"lng":[9.609,9.607,9.597,9.609],"lat":[41.082,41.071,41.075,41.082]}],[{"lng":[9.471,9.489,9.469,9.433,9.471],"lat":[41.246,41.22,41.168,41.189,41.246]}],[{"lng":[9.417,9.438,9.381,9.377,9.417],"lat":[41.269,41.214,41.21,41.234,41.269]}],[{"lng":[9.339,9.36,9.348,9.333,9.339],"lat":[41.254,41.244,41.229,41.236,41.254]}]]],null,"Countries",{"interactive":true,"className":"","stroke":true,"color":"#008000","weight":1,"opacity":0.5,"fill":true,"fillColor":"#008000","fillOpacity":0.2,"smoothFactor":0,"noClip":false},null,null,["France :  5  pizzerias","Ireland :  5  pizzerias","Italy :  1  pizzerias"],{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null]},{"method":"addLegend","args":[{"colors":["trasparent"],"labels":["Giovanni Pietro Vitali - giovannipietrovitali@gmail.com"],"na_color":null,"na_label":"NA","opacity":0.5,"position":"topright","type":"unknown","title":"Pizza Map (Learn R to make maps):","extra":null,"layerId":null,"className":"info legend","group":null}]},{"method":"addMiniMap","args":[null,null,"bottomleft",150,150,19,19,-5,false,false,false,false,false,false,{"color":"#ff7800","weight":1,"clickable":false},{"color":"#000000","weight":1,"clickable":false,"opacity":0,"fillOpacity":0},{"hideText":"Hide MiniMap","showText":"Show MiniMap"},[]]},{"method":"addLayersControl","args":[[],["Pizzerias","Countries"],{"collapsed":true,"autoZIndex":true,"position":"topright"}]}],"limits":{"lat":[35.494,55.435],"lng":[-10.618,18.517]}},"evals":[],"jsHooks":[]}</script>
<script type="application/htmlwidget-sizing" data-for="htmlwidget-066bb86a2665d8e67d60">{"viewer":{"width":"100%","height":400,"padding":0,"fill":true},"browser":{"width":"100%","height":400,"padding":0,"fill":true}}</script>
</body>
</html>
