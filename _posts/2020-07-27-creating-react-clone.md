---
layout: post
title: "Creating a React-like library from scratch"
category: blog
---

I have been developing React apps for quite a while and not a day passes by when I wonder
how the React library was made. Today, I will turn this curiosity into an adventure.

The main goal of React is to create user interfaces, and that's what we will try to achieve. We won't be doing all the fancy stuff that React does, and for a good reason. We will stick
to our goal without getting into optimizations and the nitty gritty details which will probably overwhelm anyone trying to grasp the basic stuff, including me!

Additionally, we won't *just* develop an abstract React library. We will develop the famous
todo list and you will see the React-like library getting carved out of it! Oof, that sounds fun!

So let's get to it!

### Rendering an element

How do you display an html element through JavaScript? We need the integral part of
the web for that: The JavaScript DOM API. Using it, let's display an `h1` element in the following page:

```html
<html>
    <body>
        <div id="root"></div>
    </body>
</html>
```

If you've been developing React apps, this markup looks similar. `div#root` is where React renders
your whole app. We will do the same. The JavaScript code to render `h1` looks like this:

```javascript
const root = document.getElementById("root");
const h1 = document.createElement("h1");
const hTxt = document.createTextNode("Todo List");
h1.appendChild(hTxt);
root.appendChild(h1);
```

Upon running this code, you will see the `h1` has rendered:

<iframe src="/assets/demos/creating-react-clone/v1.html" width="500px"></iframe>

As you see, we needed 5 lines of code to display/render a simple heading. We need to do the same
for other elements of a todo app i.e. the input and the list. We could do it but this is really tedious and as you may imagine, the code gets harder to maintain as your site grows. And let's remember, our react-like library should not just work for this todo app! We will abstract out these
details into a separate function called `render`.

### The `render` function

What should it look like? It could take in a representation of our element (let's call it a component) and output a DOM structure which could then be rendered in `div#root`:

```javascript
function render(component) {
    // converts `component` into DOM node
}

const componentDOM = render(...); // we'll see what goes in
root.appendChild(componentDOM);
```

The `component` will be a simple object telling us what to render. We can create functions which will return these "components". This lets us pass any additional info (think of `props`!) and return the component based on that info. In case of the header:

```javascript
function Header(props){
    return {
        name: "h1",
        children: ["Todo List"]
    };
}
```

Is it starting to look familiar? `name` is element's tag name and `children` is an array of things you want to put inside that element, which could be a simple string (for example "Todo List") or another component with a similar structure.

Our `render` function will create DOM node(s) from this kind of component so let's implement the basic steps:

```javascript
function render(component) {
    const {name, children} = component;

    // Create HTML element
    const el = document.createElement(name);

    // Iterate through children and append them in `el`
    for(let i = 0, l = children.length; i < l; i++) {
        const child = children[i];

        // Create text node
        const textNode = document.createTextNode(child);

        // Append it to `el`
        el.appendChild(textNode);
    }

    return el;
}
```

You probably figured out what goes in the render function. This shall produce the same result as in the last example:

```javascript
const componentDOM = render(Header());
root.appendChild(componentDOM);
```

We're far from done here. There are some basic limitations of our render function right now:

1. It does not take into account other HTML attributes like `id`, `class`, `style` etc
2. It does not add event handlers onto an element
3. It cannot render complex components beside strings

There are more but these are what come into one's mind right off the bat. We will fix them gradually as we progress.

### Adding the input element

We should be able to add new todos in our yet-to-develop todo list. So I will create an `input` element. We will create this inside a `form` element since we want to add new todos on form submission. Our form component is going to be a little more complex since this time, we not only have event handlers but also child components (and not just mere strings!). Here it is:

```javascript
function TodoForm(props) {
    const inputEl = {
        name: "input",
        type: "text",
        placeholder: "Add a new todo",
        className: "input-todo",
        style: {
            padding: "6px 12px",
            display: "block",
            width: "100%",
            "box-sizing": "border-box"
        }
    };
    return {
        name: "form",
        style: {
            width: "50%",
            margin: "auto"
        },
        onSubmit: props.onSubmit,
        children: [inputEl]
    };
}
```

Notice we have things now on which the `render` function won't work. Let's refactor it so it supports our `TodoForm` component. There will be so many attributes like `placeholder` and `onSubmit` so it has to be as generic as possible.

```javascript
function render(component) {
    // We collect all other attributes in `otherAttributes`
    const {name, style, children, ...otherAttributes} = component;

    const el = document.createElement(name);

    // Add all styles to the element
    if (typeof style === "object") {
        for (const styleName in style) {
            el.style[styleName] = style[styleName];
        }
    }

    // Add all attributes to the element
    // An attribute can be a simple string or an event handler
    // If it is an event handler, it needs to be DOM supported so we'll check it
    // against our list of possible events
    for(let key in otherAttributes) {
        const value = otherAttributes[key];
        if (typeof value === "string") {
            // class is a reserved keyword so we'll instead use "className" in our
            // components and convert it back to "class" here before setting it.
            if (key === "className") {
                key = "class";
            }
            el.setAttribute(key, value);
            continue;
        }
        if (typeof value === "function" && eventsMap[key]) {
            el.addEventListener(eventsMap[key], value);
        }
    }

    // If there are children, iterate through them and append them in `el`
    if (Array.isArray(children)) {
        for(let i = 0, l = children.length; i < l; i++) {
            const child = children[i];

            // it's text, create a text node
            if (typeof child === "string") {
                const textNode = document.createTextNode(child);
                el.appendChild(textNode);
                continue;
            }
    
            // It's a component; we need to call render again
            if (typeof child === "object") {
                el.appendChild(render(child));
            }
        }
    }

    return el;
}

```

We have taken care of the limitations we previously discussed. The event handler names in components are prefixed with "on" (`onSubmit` for example), but when using `addEventListener`, that's an invalid event name. So we'll map these `onEvent` keys to just `event`. The key map will look something like this:

```javascript
const eventsMap = {
    onSubmit: "submit",
    onClick: "click",
    // and so on ...
};
```

If you haven't noticed, the `render` function is now recursive! If we've a component as a child of another component, we know it is going to have a similar structure to its parent component, so we can just call render on that again.

We now need to render these two components: `Header` and `TodoForm`. We can introduce another component `App` which will contain both of these components and will be rendered at `div#root`.

### The `App` component

The app component is fairly simple, for now. It contains `Header` and `TodoForm` as its children. Additionally, it passes the `onSubmit` handler down to our form component. Let's code that:

```javascript
function App() {
    function onSubmit(e) {
        e.preventDefault();
        const inputEl = e.target.querySelector(".input-todo");
        console.log(inputEl.value);
    }
    return {
        name: "div",
        className: "app",
        id: "app",
        children: [
            Header(),
            TodoForm({ onSubmit })
        ]
    }
}
```

Before implementing the `TodoList` component, let's run this code and be confident of ourselves. Try typing something in the form and submitting it (just press Enter), you'll see the input getting logged in the console.

<iframe src="/assets/demos/creating-react-clone/v2.html" width="500px"></iframe>
*We added some styles to h1, form and input so everything looks nice and centered*

The next step is developing the `TodoList` component.

### The `TodoList` component

This is going to be a `ul` element with todo items in each `li`. The todo items will look like this:

```javascript
let items = [
    {
        id: 0,
        text: "Todo 1"
    },
    {
        id: 1,
        text: "Todo 2"
    }
]
```

We need to be able to delete these todo items as well, which is one reason we've added `id` in each item. We'll pass these items as a prop to `TodoList` which will be the last child of the `App` component:

```javascript
function App() {
    // ...
    function onRemoveTodo(id) {
        // ...
    }

    return {
        // ...
        children: [
            // ...
            TodoList({ items, onRemoveTodo })
        ]
    }
}
```

Let's implement the list component:

```javascript
function TodoList(props) {
    const { items, onRemoveTodo } = props;
    return {
        name: "ul",
        className: "todo-list",
        children: items.map(item => TodoItem({ ...item, onRemoveTodo }))
    }
}
```

We've mapped the items to a list of another components (`TodoItem`s) so let's implement that as well:

```javascript
function TodoItem(props) {
    const { id, text, onRemoveTodo } = props;
    const removeBtn = {
        name: "button",
        onClick: () => onRemoveTodo(id),
        children: ["Remove"]
    };

    return {
        name: "li",
        children: [text, removeBtn]
    }
}
```

Before implementing `onSubmit` and `onRemoveTodo`, let's run this code:
<iframe src="/assets/demos/creating-react-clone/v3.html" width="500px" height="200px"></iframe>

### Rendering on updates

What we have done so far is static rendering, that is, what we see being displayed on browser will remain static and not change. That's a big limitation. Our app should be able to respond (or re-render) on updates, whether they are a result of changing props or some state. In our case, we donot have anything similar to React state. We do have props, but they won't result in re-rendering upon a change.

We won't introduce state as of yet, neither we will create a generic way to re-render on props change. Instead, just to see our final product, we will re-render the `TodoList` component in our implementatations of `onSubmit` and `onRemoveTodo` event handler. So let's do that:

```javascript
function App() {
    function onSubmit(e) {
        e.preventDefault();
        const inputEl = e.target.querySelector(".input-todo");
        const lastItem = items[items.length - 1];
        const newId = lastItem ? lastItem.id + 1: 0;
        items.push({ id: newId, text: inputEl.value });

        // We've access to our rendered app so we'll just use that
        app.removeChild(root.querySelector(".todo-list"));
        app.appendChild(render(TodoList({ items, onRemoveTodo })));

        // Clear the input
        inputEl.value = "";
    }
    function onRemoveTodo(id) {
        const index = items.findIndex(item => item.id == id);
        items.splice(index, 1);

        app.removeChild(root.querySelector(".todo-list"));
        app.appendChild(render(TodoList({ items, onRemoveTodo })));
    }
    return {
        // ...
    }
}
```

In both of our handlers, we are simply removing `.todo-list` and rendering the whole `TodoList` component again! Not so good! Firstly, this is too tightly coupled with the `App` component so we would have to do something similar for every component if it's not just a Todo app as simple as ours. That's a mess. Secondly, we are rendering all the items again which is not efficient at all.

Let's ignore those problems and enjoy the view for now:

<iframe src="/assets/demos/creating-react-clone/v4.html" width="500px" height="200px"></iframe>

It works! You can now add and remove todo items! We've come a long way but there's still a lot to do. We'll do the remaining work in part 2 of this article.

I hope you liked it. If you did and would like to see Part 2 as well, follow me on <a href="https://twitter.com/farooqar" target="_blank">Twitter</a> where I tweet about my upcoming posts as well as other good stuff!