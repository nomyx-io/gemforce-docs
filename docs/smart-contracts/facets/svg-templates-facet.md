# SVGTemplatesFacet

The `SVGTemplatesFacet` is a specialized component within the Gemforce Diamond smart contract system, designed to manage and render on-chain Scalable Vector Graphics (SVG) templates. This facet enables dynamic, customizable visual representations of NFTs and other on-chain assets without relying on off-chain image hosting.

## Purpose

The primary purpose of the `SVGTemplatesFacet` is to provide a powerful and flexible mechanism for generating dynamic artwork directly from smart contract data. By storing SVG templates and allowing for programmatic substitution of variables, it ensures that NFT metadata and visuals are fully decentralized, immutable, and responsive to on-chain state changes. This is particularly useful for generative art, dynamic collectibles, or any digital asset whose visual representation evolves with its attributes.

## Key Features

*   **On-Chain SVG Storage**: Securely stores base SVG templates directly on the blockchain.
*   **Variable Substitution**: Allows defining placeholders within SVG templates that can be dynamically replaced with on-chain data (e.g., token ID, attributes, owner balance).
*   **Template Management**: Functions for adding, updating, and removing SVG templates by authorized entities.
*   **Dynamic Rendering**: Provides a function to generate a complete SVG image by applying specific data to a chosen template.
*   **Event Emission**: Emits events for template management and rendering operations, providing transparency and traceability.

## Functions

### `addTemplate(uint256 _templateId, string calldata _svgContent)`

Adds a new SVG template to the facet.

*   **Parameters**:
    *   `_templateId` (uint256): A unique identifier for the SVG template.
    *   `_svgContent` (string calldata): The raw SVG content (string) with placeholders for variables.
*   **Requirements**:
    *   Only the Diamond owner can call this function.
    *   `_templateId` must be unique and not zero.
    *   `_svgContent` should be valid SVG XML.
*   **Emits**: `TemplateAdded(uint256 indexed templateId, string svgContent)`

### `updateTemplate(uint256 _templateId, string calldata _newSvgContent)`

Updates an existing SVG template.

*   **Parameters**:
    *   `_templateId` (uint256): The ID of the template to update.
    *   `_newSvgContent` (string calldata): The updated SVG content.
*   **Requirements**:
    *   Only the Diamond owner can call this function.
    *   `_templateId` must correspond to an existing template.
*   **Emits**: `TemplateUpdated(uint256 indexed templateId, string newSvgContent)`

### `removeTemplate(uint256 _templateId)`

Removes an SVG template from the facet.

*   **Parameters**:
    *   `_templateId` (uint256): The ID of the template to remove.
*   **Requirements**:
    *   Only the Diamond owner can call this function.
    *   `_templateId` must correspond to an existing template.
*   **Emits**: `TemplateRemoved(uint256 indexed templateId)`

### `renderSVG(uint256 _templateId, string[] calldata _keys, string[] calldata _values)`

Renders a complete SVG string by substituting variables in a given template.

*   **Parameters**:
    *   `_templateId` (uint256): The ID of the SVG template to use.
    *   `_keys` (string[] calldata): An array of placeholder keys (e.g., `"{NAME}"`, `"{COLOR}"`)
    *   `_values` (string[] calldata): An array of corresponding values to substitute.
*   **Requirements**:
    *   The length of `_keys` must match the length of `_values`.
    *   `_templateId` must correspond to an existing template.
*   **Returns**: (string) The fully rendered SVG string.

## Events

### `TemplateAdded(uint256 indexed templateId, string svgContent)`

Emitted when a new SVG template is successfully added.

*   **Parameters**:
    *   `templateId` (uint256): The ID of the template.
    *   `svgContent` (string): The content of the added SVG.

### `TemplateUpdated(uint256 indexed templateId, string newSvgContent)`

Emitted when an SVG template is successfully updated.

*   **Parameters**:
    *   `templateId` (uint256): The ID of the updated template.
    *   `newSvgContent` (string): The new content of the SVG.

### `TemplateRemoved(uint256 indexed templateId)`

Emitted when an SVG template is successfully removed.

*   **Parameters**:
    *   `templateId` (uint256): The ID of the removed template.

## Usage Example

```solidity
// Assuming 'diamond' is an instance of IDiamond and 'owner' is the diamond owner

// Example SVG template with placeholders
string memory baseSvg = "<svg viewBox="0 0 100 100"><circle cx="50" cy="50" r="{RADIUS}" fill="{COLOR}"/><text x="10" y="90">ID: {TOKEN_ID}</text></svg>";

// Add a new SVG template
uint256 templateId = 1;
IDiamond(diamond).addTemplate(templateId, baseSvg);

// Define keys and values for substitution
string[] memory keys = new string[](3);
keys[0] = "{RADIUS}";
keys[1] = "{COLOR}";
keys[2] = "{TOKEN_ID}";

string[] memory values = new string[](3);
values[0] = "40";
values[1] = "blue";
values[2] = "12345";

// Render the SVG
string memory renderedSvg = IDiamond(diamond).renderSVG(templateId, keys, values);

// renderedSvg would contain "<svg viewBox="0 0 100 100"><circle cx="50" cy="50" r="40" fill="blue"/><text x="10" y="90">ID: 12345</text></svg>"