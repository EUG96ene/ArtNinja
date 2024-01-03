# 🎨 NinjaSketch

I'm building an Excalidraw clone with React and TypeScript. For the sketch-like designs, rough.js is being utilized. Rough.js is used for the sketchy, hand-drawn style, similar to what you see in Excalidraw.

The primary purpose of this project is for learning and educational purposes.

More updates coming soon...

## 🎬 Behind the Scenes: Building NinjaSketch

A step by step guide on how I created this. The code is often changed as I'm always adjusting it for the best results. 🔮

<details> 
<summary><h3> 1️⃣ Rendering canvas with rough.js </h3> </summary>

In the `useLayoutEffect`, I first grab the canvas from the webpage and prepare it for drawing. I'm doing this because I don't want old sketches to mix with the new one, ensuring a clean and clear drawing every time.

It clears any previous drawings to start fresh. Then, I use rough.js to make the drawings look sketchy and hand-drawn.

A rectangle is drawn on this prepared canvas. All of this is done before the browser updates the display, which means the drawing appears all at once.

```javascript
import { useLayoutEffect } from "react";
import rough from "roughjs";

export default function App() {
  useLayoutEffect(() => {
    const canvas = document.getElementById("canvas") as HTMLCanvasElement;
    const context = canvas.getContext("2d") as CanvasRenderingContext2D;
    context.clearRect(0, 0, canvas.width, canvas.height);

    const roughCanvas = rough.canvas(canvas);
    const rect = roughCanvas.rectangle(10, 10, 200, 200);
    roughCanvas.draw(rect);
  });

  return (
    <div>
      <canvas id="canvas" width={window.innerWidth} height={window.innerHeight}>
        Canvas
      </canvas>
    </div>
  );
}
```

</details>
<details>
<summary><h3>2️⃣ Drawing the canvas</h3> </summary>

When I press the mouse down, the `handleMouseDown` function activates. It indicates I'm starting to draw by setting the `drawing` state to true. This means I'm beginning a new shape right where my cursor is at. The shape I draw, a line or rectangle, is decided by my previous choice and tracked by the `elementType` state, and the radio buttons let me switch between lines and rectangles.

While I move the mouse, the `handleMouseMove` function activates. If I'm drawing, the shape follows my cursor.

On the technical side, I find the last drawing I started with `const index = elements.length - 1;`. I then capture my mouse's current position with `const { clientX, clientY } = event;`. The `const { x1, y1 } = elements[index];` gets the starting point of my current shape, basically marking the first corner or line end. Using the initial and current positions, I update the shape I'm drawing with `const updateElement = createElement(x1, y1, clientX, clientY);`. Next, I make a copy of all my drawings and update the most recent one, the shape I'm currently changing, with the new version. This updated collection is then saved back into the `elements` state.

The drawing stops when I release the mouse, which the `handleMouseUp` function handles, ending the drawing.

I store every stroke and shape in an array, which is the `elements` state, and `useLayoutEffect` redraws the canvas with each new addition.

The clear button empties the array for a fresh canvas.

```javascript
import { MouseEvent, useLayoutEffect, useState } from "react";
import rough from "roughjs";

type ElementType = {
  x1: number;
  y1: number;
  x2: number;
  y2: number;
  // TODO: add type
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  roughElement: any;
};

export default function App() {
  const [elements, setElements] = useState<ElementType[]>([]);
  const [drawing, setDrawing] = useState(false);
  const [elementType, setElementType] = useState<"line" | "rectangle">("line");

  const generator = rough.generator();

  const createElement = (
    x1: number,
    y1: number,
    x2: number,
    y2: number
  ): ElementType => {
    const roughElement =
      elementType === "line"
        ? generator.line(x1, y1, x2, y2)
        : generator.rectangle(x1, y1, x2 - x1, y2 - y1);
    return { x1, y1, x2, y2, roughElement };
  };

  useLayoutEffect(() => {
    const canvas = document.getElementById("canvas") as HTMLCanvasElement;
    const context = canvas.getContext("2d") as CanvasRenderingContext2D;
    context.clearRect(0, 0, canvas.width, canvas.height);
    const roughCanvas = rough.canvas(canvas);
    elements.forEach(({ roughElement }) => {
      roughCanvas.draw(roughElement);
    });
  }, [elements]);

  const handleMouseDown = (event: MouseEvent<HTMLCanvasElement>) => {
    setDrawing(true);
    const { clientX, clientY } = event;
    const element = createElement(clientX, clientY, clientX, clientY);
    setElements((prevState) => [...prevState, element]);
  };

  const handleMouseMove = (event: MouseEvent<HTMLCanvasElement>) => {
    if (!drawing) {
      return;
    }
    const index = elements.length - 1;
    const { clientX, clientY } = event;
    const { x1, y1 } = elements[index];
    const updateElement = createElement(x1, y1, clientX, clientY);
    const elementsCopy = [...elements];
    elementsCopy[index] = updateElement;
    setElements(elementsCopy);
  };

  const handleMouseUp = () => {
    setDrawing(false);
  };
  return (
    <div>
      <div style={{ position: "fixed" }}>
        <button onClick={() => setElements([])}>Clear</button>
        <input
          type="radio"
          name="line"
          id="line"
          checked={elementType === "line"}
          onChange={() => setElementType("line")}
        />
        <label htmlFor="line">line</label>
        <input
          type="radio"
          name="rectangle"
          id="rectangle"
          checked={elementType === "rectangle"}
          onChange={() => setElementType("rectangle")}
        />
        <label htmlFor="rectangle">rectangle</label>
      </div>
      <canvas
        id="canvas"
        width={window.innerWidth}
        height={window.innerHeight}
        onMouseDown={handleMouseDown}
        onMouseUp={handleMouseUp}
        onMouseMove={handleMouseMove}
      >
        Canvas
      </canvas>
    </div>
  );
}
```

</details>

<details>
<summary><h3>3️⃣ Moving Elements</h3></summary>

I've renamed `elementType` and `setElementType` to `tools` and `setTools` to make it clearer. Now, I pick from different tools using radio buttons, not just setting an element type.

I've renamed `setDrawing` and `drawing` to `setAction` and `action` for more generic use. Now, I can do things like move elements if the tool is `selection` and the action is `moving`. This lets me move what I've drawn, making it more interactive.

An enum for `Tools` has been created, making it clearer and more organized to switch between "selection", "line", and "rectangle" tools.

`getElementAtPosition` finds the element at the cursors position, so I know which shape you're trying to move.

`isWithinElement` function figures out if I can move a shape with my cursor.

For rectangles, it checks if the cursor is inside the shapes edges like this:

```javascript
if (type === Tools.Rectangle) {
  const minX = Math.min(x1, x2);
  const maxX = Math.max(x1, x2);
  const minY = Math.min(y1, y2);
  const maxY = Math.max(y1, y2);
  return x >= minX && x <= maxX && y >= minY && y <= maxY;
}
```

<img src='./public/rectangle.png' />

For lines, it checks if the cursor is close to the line by measuring distances:

```javascript
else {
  const a = { x: x1, y: y1 };
  const b = { x: x2, y: y2 };
  const c = { x, y };
  const offset = distance(a, b) - (distance(a, c) + distance(b, c));
  return Math.abs(offset) < 1;
}
```

<img src='./public/line.png' />

So if the cursor is almost as far from the lines ends as the line is long, it's "on" the line.

I learned the line method from [stack overflow](https://stackoverflow.com/questions/17692922/check-is-a-point-x-y-is-between-two-points-drawn-on-a-straight-line/17693146#17693146).

The `distance` function just helps me find out how far apart two points are.

```javascript
import { MouseEvent, useLayoutEffect, useState } from "react";
import rough from "roughjs";

type ElementType = {
  id: number;
  x1: number;
  y1: number;
  x2: number;
  y2: number;
  type: Tools;
  // TODO: add type
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  roughElement: any;
  offsetX?: number;
  offsetY?: number;
};

enum Tools {
  Selection = "selection",
  Line = "line",
  Rectangle = "rectangle",
}

export default function App() {
  const [elements, setElements] = useState<ElementType[]>([]);
  const [action, setAction] = useState("none");
  const [tool, setTool] = useState<Tools>(Tools.Line);
  const [selectedElement, setSelectedElement] = useState<ElementType | null>();
  const generator = rough.generator();

  const createElement = (
    id: number,
    x1: number,
    y1: number,
    x2: number,
    y2: number,
    type: Tools
  ): ElementType => {
    const roughElement =
      type === Tools.Line
        ? generator.line(x1, y1, x2, y2)
        : generator.rectangle(x1, y1, x2 - x1, y2 - y1);
    return { id, x1, y1, x2, y2, type, roughElement };
  };

  type Point = { x: number; y: number };

  const distance = (a: Point, b: Point) =>
    Math.sqrt(Math.pow(a.x - b.x, 2) + Math.pow(a.y - b.y, 2));

  const isWithinElement = (x: number, y: number, element: ElementType) => {
    const { type, x1, y1, x2, y2 } = element;

    if (type === Tools.Rectangle) {
      const minX = Math.min(x1, x2);
      const maxX = Math.max(x1, x2);
      const minY = Math.min(y1, y2);
      const maxY = Math.max(y1, y2);
      return x >= minX && x <= maxX && y >= minY && y <= maxY;
    } else {
      const a = { x: x1, y: y1 };
      const b = { x: x2, y: y2 };
      const c = { x, y };
      const offset = distance(a, b) - (distance(a, c) + distance(b, c));
      return Math.abs(offset) < 1;
    }
  };

  const getElementAtPosition = (
    x: number,
    y: number,
    elements: ElementType[]
  ) => {
    return elements.find((element) => isWithinElement(x, y, element));
  };

  useLayoutEffect(() => {
    const canvas = document.getElementById("canvas") as HTMLCanvasElement;
    const context = canvas.getContext("2d") as CanvasRenderingContext2D;
    context.clearRect(0, 0, canvas.width, canvas.height);

    const roughCanvas = rough.canvas(canvas);

    elements.forEach(({ roughElement }) => {
      roughCanvas.draw(roughElement);
    });
  }, [elements]);

  const updateElement = (
    id: number,
    x1: number,
    y1: number,
    x2: number,
    y2: number,
    type: Tools
  ) => {
    const updateElement = createElement(id, x1, y1, x2, y2, type);

    const elementsCopy = [...elements];
    elementsCopy[id] = updateElement;
    setElements(elementsCopy);
  };

  const handleMouseDown = (event: MouseEvent<HTMLCanvasElement>) => {
    const { clientX, clientY } = event;

    if (tool === Tools.Selection) {
      const element = getElementAtPosition(clientX, clientY, elements);
      if (element) {
        const offsetX = clientX - element.x1;
        const offsetY = clientY - element.y1;
        setSelectedElement({ ...element, offsetX, offsetY });
        setAction("moving");
      }
    } else {
      const id = elements.length;
      const element = createElement(
        id,
        clientX,
        clientY,
        clientX,
        clientY,
        tool
      );
      setElements((prevState) => [...prevState, element]);
      setAction("drawing");
    }
  };

  const handleMouseMove = (event: MouseEvent<HTMLCanvasElement>) => {
    const { clientX, clientY } = event;

    if (tool === Tools.Selection) {
      (event.target as HTMLElement).style.cursor = getElementAtPosition(
        clientX,
        clientY,
        elements
      )
        ? "move"
        : "default";
    }

    if (action === "drawing") {
      const index = elements.length - 1;
      const { x1, y1 } = elements[index];
      updateElement(index, x1, y1, clientX, clientY, tool);
    } else if (action === "moving" && selectedElement) {
      const { id, x1, x2, y1, y2, type, offsetX, offsetY } = selectedElement;
      const safeOffsetX = offsetX ?? 0;
      const safeOffsetY = offsetY ?? 0;
      const newX1 = clientX - safeOffsetX;
      const newY1 = clientY - safeOffsetY;
      // 🫐 Calculate the new position for x2 and y2 based on the original size
      const newX2 = newX1 + (x2 - x1);
      const newY2 = newY1 + (y2 - y1);

      updateElement(id, newX1, newY1, newX2, newY2, type);
    }
  };

  const handleMouseUp = () => {
    setAction("none");
  };

  return (
    <div>
      <div style={{ position: "fixed" }}>
        <button onClick={() => setElements([])}>Clear</button>

        <input
          type="radio"
          name="selection"
          id="selection"
          checked={tool === Tools.Selection}
          onChange={() => setTool(Tools.Selection)}
        />
        <label htmlFor="selection">selection</label>
        <input
          type="radio"
          name="line"
          id="line"
          checked={tool === Tools.Line}
          onChange={() => setTool(Tools.Line)}
        />
        <label htmlFor="line">line</label>

        <input
          type="radio"
          name="rectangle"
          id="rectangle"
          checked={tool === Tools.Rectangle}
          onChange={() => setTool(Tools.Rectangle)}
        />

        <label htmlFor="rectangle">rectangle</label>
      </div>
      <canvas
        id="canvas"
        width={window.innerWidth}
        height={window.innerHeight}
        onMouseDown={handleMouseDown}
        onMouseUp={handleMouseUp}
        onMouseMove={handleMouseMove}
      >
        Canvas
      </canvas>
    </div>
  );
}
```

</details>
