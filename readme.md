# Ashby Design an API for a React Component Challenge

Jumping right in, implemented as a compound component. With the most generic use:

```jsx
<Autocomplete
  results={autocompleteResults}
  onSearchValueChange={setSearchValue}
>
  <Autocomplete.ButtonTrigger /> // Or `DropdownTrigger`
  <Autocomplete.Panel>
    <Autocomplete.SearchBox /> // Optional
    <Autocomplete.ResultsList />
    <Autocomplete.AddItemButton /> // Optional
  </Autocomplete.Panel>
</Autocomplete>
```

This approach enables less experienced React devs to:

1. Easily browse child components with IDE lookup on `<Autocomplete.[...]`
2. Achieve a semantic autocomplete without custom UI code.

And more experienced React devs to:

1. Override (or add) any element by using the render prop - avoiding polluting with the `Autocomplete` context
2. Piggy-back events like `onClick` for e.g. analytics without interfering with `Autocomplete` logic

Breaking down the elements:

## `<Autocomplete>`

`Autocomplete` exposes its children and a Context to enable it to operate without prop drilling / knowing the internal API.

```jsx
<Autocomplete
	results=[Array<string | { id: string, value: string, category?: string, metadata: unknown }]
	// Assumption made that categorised results would be returned in the shape above ^
	onSearchValueChange=[(searchValue: string) => void]
	onSelectedResultChange=[(resultId: string) => void]
	onAddItem=[(searchValue: string)} => void]
>
	context = {
		`results`,
		`searchValue`, `setSearchValue`,
		`selectedResult`, `setSelectedResult`,
		`triggerRef`,
		`isPanelOpen`, `setIsPanelOpen`,
		`onAddItem`
	}
	return typeof children === 'function' ? children(context) : children
	// As an escape hatch, if `children` is a function, the context is passed through
	// to enable arbitrary components/overrides.
</Autocomplete>
```

Children as a function would be used to achieve a custom trigger (for example):

```jsx
<Autocomplete ...>
	{({ isPanelOpen, setIsPanelOpen }) => (
		<button onClick={() => setIsPanelOpen(!isPanelOpen}>Arbitrary trigger</button>
		...
	)}
</Autocomplete>
```

When `setSearchValue`, `setSelectedResult` or `onAddItem` are called, the parent callbacks are triggered.

## `<Autocomplete.ButtonTrigger` and `<Autocomplete.DropdownTrigger>`

Both are just encapsulations of calling `setIsPanelOpen` with display differences. They forward a ref to their outermost element (`triggerRef`) so `Panel` can render anchored to their positions.

```jsx
<Autocomplete.ButtonTrigger children=[JSX] ...ButtonProps>
	<Button {...ButtonProps} onClick={(e) => { onClick?.(e); setIsPanelOpen(bool); }}>
		{children}
	</Button>
</Autocomplete.ButtonTrigger>

<Autocomplete.DropdownTrigger placeholder=[string]>
	<button {...}>
		{selectedValue ?? placeholder}
		<ChevronSVG />
	</button>
</Autocomplete.DropdownTrigger>
```

`DropdownTrigger` returns a button that _looks like_ a select element. It can use `selectedValue` to show the value like a select. As a nice detail it could use `isPanelOpen` to rotate the chevron.

## `<Autocomplete.Panel>`

Wrapper component rendered in a Portal to circumvent stacking context / overflows. Positioned based on `triggerRef`. Would consider a library like `floating-ui` to avoid writing visibility logic. Returns `null` when panel is closed.

```jsx
<Autocomplete.Panel>
	{isPanelOpen ? <Portal style=[triggerRef.getBoundingClientRect...]>{children}</Portal> : null}
</Autocomplete.Panel>
```

## `<Autocomplete.SearchBox>`

```jsx
<Autocomplete.SearchBox ...TextInputProps>
	<TextInput {...TextInputProps} value=[searchValue] onChange=[setSearchValue]>
</Autocomplete.SearchBox>
```

## `<Autocomplete.ResultsList>`

If the results are a string array, render a `ResultButton` for each. Otherwise for categorised results, convert into an object of arrays keyed by `category`.

Iterate over the keys and return a `CategoryHeading` for each with `ResultButton`s within. Clicking triggers `setSelectedResult`.

```jsx
<Autocomplete.ResultsList>
	//For each category
	<Autocomplete.CategoryHeading>
		{category}
	</Autocomplete.CategoryHeading>
	//For each result..
	<Autocomplete.ResultButton onClick=[setSelectedResult]>
		{value}
	</Autocomplete.ResultButton>
</Autocomplete.ResultsList>
```

## `<Autocomplete.AddItemButton>`

```jsx
<button onClick=[onAddItem(searchValue)]>Add an item</button>
```

## Assumptions / closing thoughts

API design influenced by using https://headlessui.com/react/combobox on several projects. I'm a fan of the balance they strike between sensible defaults and escape hatches.

Accessibility-wise I'd be thinking about auto focus on the panel when it opens to ensure it remains keyboard navigable. Whether or not to tackle next/previous selection using arrow keys would be something to consider as well.

For UX improvements I'd add an event listener for closing the panel on click-off and pressing the escape key (like a modal).

Depending on the query implementation, it may also make sense to provide an adjustable debounce.
