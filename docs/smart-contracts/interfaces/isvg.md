# ISVG Interface

The `ISVG` interface defines the standard for smart contracts that generate and manage Scalable Vector Graphics (SVG) on-chain. This interface is crucial for creating dynamic and algorithmically generated NFT art, badges, or other visual assets directly from smart contracts within the Gemforce ecosystem.

## Overview

`ISVG` provides:

- **SVG Generation**: Functions to generate complete SVG strings based on input parameters.
- **Template Management**: Mechanisms for managing and combining SVG templates.
- **Render Parameterization**: Allow for dynamic data to influence SVG output.
- **Data Encoding**: Efficiently encode visual data on-chain.

## Key Features

### SVG Rendering
- **`renderSVG()`**: Main function to produce a final SVG string.
- **Dynamic Elements**: Support for injecting data (colors, shapes, text, etc.) into templates.
- **Component Assembly**: Combine smaller SVG components into a larger scene.

### Template and Asset Management
- **Store Templates**: On-chain storage for SVG fragments or predefined layers.
- **Reference Assets**: Ability to reference other on-chain or off-chain assets (e.g., fonts, images encoded as base64).
- **Template Composition**: Logic to mix and match predefined SVG elements.

### Metadata Integration
- Can be used to generate SVG content that forms the `image` field in ERC721 metadata URIs.

## Interface Definition

```solidity
interface ISVG {
    // Events
    event SVGGenerated(
        uint256 indexed tokenId,
        string indexed svgType,
        uint256 indexed version,
        string svgDataURI
    );
    event TemplateAdded(
        string indexed templateName,
        uint256 indexed version,
        address indexed creator
    );
    event TemplateUpdated(
        string indexed templateName,
        uint256 indexed version,
        address indexed updater
    );
    event FragmentAdded(
        string indexed fragmentName,
        uint256 indexed version,
        address indexed creator
    );
    event FragmentUpdated(
        string indexed fragmentName,
        uint256 indexed version,
        address indexed updater
    );

    // Structs
    struct SVGTemplate {
        string name;
        uint256 version;
        string content; // Base SVG structure, often with placeholders
        bool active;
        address creator;
        uint256 createdAt;
    }

    struct SVGFragment {
        string name;
        uint256 version;
        string content; // Reusable SVG snippets
        address creator;
        uint256 createdAt;
    }

    // Core SVG Generation Functions
    function renderSVG(
        string calldata templateName,
        uint256 templateVersion,
        bytes calldata data
    ) external view returns (string memory svgURI);

    function getRenderedSVG(uint256 tokenId) external view returns (string memory svgURI);

    // Template Management Functions
    function addTemplate(
        string calldata name,
        uint256 version,
        string calldata content
    ) external;

    function updateTemplate(
        string calldata name,
        uint256 version,
        string calldata newContent
    ) external;

    function getTemplate(
        string calldata name,
        uint256 version
    ) external view returns (SVGTemplate memory);

    function deactivateTemplate(string calldata name, uint256 version) external;

    // Fragment Management Functions
    function addFragment(
        string calldata name,
        uint256 version,
        string calldata content
    ) external;

    function updateFragment(
        string calldata name,
        uint256 version,
        string calldata newContent
    ) external;

    function getFragment(
        string calldata name,
        uint256 version
    ) external view returns (SVGFragment memory);

    // View Functions for Listing
    function getTemplateNames() external view returns (string[] memory);
    function getAvailableVersions(string calldata templateName) external view returns (uint256[] memory);
    function getFragmentNames() external view returns (string[] memory);
}
```

## Core Functions

### renderSVG()
Generates and returns a data URI (typically base64-encoded) containing the complete SVG image. This function takes a template name, version, and dynamic data as input to customize the output.

**Parameters:**
- `templateName`: The name of the SVG template to use.
- `templateVersion`: The version of the template.
- `data`: ABI-encoded bytes containing dynamic parameters (e.g., colors, text, numbers) to be inserted into the SVG template.

**Returns:**
- `string`: A data URI (e.g., `data:image/svg+xml;base64,...`) containing the generated SVG.

**Usage:**
```solidity
// Example: Render an SVG for a token, passing color and text details
bytes memory renderData = abi.encode(
    "red",
    "Hello Gemforce!",
    123
);
string memory svgDataURI = svgContract.renderSVG("CryptoBadge", 1, renderData);
```

### addTemplate()
Adds a new SVG template to the contract's storage. Templates are typically base SVG structures with placeholders for dynamic data.

**Parameters:**
- `name`: A unique name for the template.
- `version`: A version number for the template.
- `content`: The raw SVG string content (with placeholders).

**Access Control:**
- Typically restricted to the contract owner or an authorized administrator.

### addFragment()
Adds a reusable SVG fragment (e.g., a common icon, a repeating pattern) to the contract's storage. Fragments can be embedded into templates.

**Parameters:**
- `name`: A unique name for the fragment.
- `version`: A version number for the fragment.
- `content`: The raw SVG string content of the fragment.

## Implementation Example

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Base64.sol";
// Optional: Using StringUtils for more complex string manipulation if not available
// import "./libraries/StringUtils.sol"; 

contract SVGGenerator is ISVG, Ownable {
    // Mapping for SVG templates: name => version => SVGTemplate
    mapping(string => mapping(uint256 => SVGTemplate)) private _templates;
    // Mapping for SVG fragments: name => version => SVGFragment
    mapping(string => mapping(uint256 => SVGFragment)) private _fragments;

    // To list available templates and fragments
    string[] private _templateNames;
    mapping(string => uint256[]) private _templateVersions;
    string[] private _fragmentNames;

    // Assuming a mapping from tokenId to its SVG data URI is maintained elsewhere or generated on the fly per request
    // This example focuses on the generation logic itself.
    mapping(uint256 => string) private _renderedSVGStorage; // For getRenderedSVG if needed for persistence

    constructor() {
        // Initial setup or placeholders can be added here
    }

    function addTemplate(
        string calldata name,
        uint256 version,
        string calldata content
    ) external override onlyOwner {
        require(bytes(name).length > 0, "Name cannot be empty");
        require(bytes(content).length > 0, "Content cannot be empty");
        require(_templates[name][version].creator == address(0), "Template already exists");

        _templates[name][version] = SVGTemplate({
            name: name,
            version: version,
            content: content,
            active: true,
            creator: msg.sender,
            createdAt: block.timestamp
        });
        
        bool nameExists = false;
        for(uint256 i=0; i<_templateNames.length; i++){
            if(keccak256(abi.encodePacked(_templateNames[i])) == keccak256(abi.encodePacked(name))){
                nameExists = true;
                break;
            }
        }
        if(!nameExists){
            _templateNames.push(name);
        }
        _templateVersions[name].push(version);

        emit TemplateAdded(name, version, msg.sender);
    }

    function updateTemplate(
        string calldata name,
        uint256 version,
        string calldata newContent
    ) external override onlyOwner {
        SVGTemplate storage templateEntry = _templates[name][version];
        require(templateEntry.creator != address(0), "Template not found");
        require(bytes(newContent).length > 0, "Content cannot be empty");

        templateEntry.content = newContent;
        templateEntry.lastUpdated = block.timestamp; // Assuming a 'lastUpdated' field in struct
        emit TemplateUpdated(name, version, msg.sender);
    }

    function deactivateTemplate(string calldata name, uint256 version) external override onlyOwner {
        SVGTemplate storage templateEntry = _templates[name][version];
        require(templateEntry.creator != address(0), "Template not found");
        require(templateEntry.active == true, "Template already inactive");
        templateEntry.active = false;
        // Optionally remove from _templateVersions or _templateNames if no active versions remain
    }

    function getTemplate(string calldata name, uint256 version) external view override returns (SVGTemplate memory) {
        SVGTemplate memory templateEntry = _templates[name][version];
        require(templateEntry.creator != address(0), "Template not found");
        return templateEntry;
    }

    function addFragment(
        string calldata name,
        uint256 version,
        string calldata content
    ) external override onlyOwner {
        require(bytes(name).length > 0, "Name cannot be empty");
        require(bytes(content).length > 0, "Content cannot be empty");
        require(_fragments[name][version].creator == address(0), "Fragment already exists");

        _fragments[name][version] = SVGFragment({
            name: name,
            version: version,
            content: content,
            creator: msg.sender,
            createdAt: block.timestamp
        });

        bool nameExists = false;
        for(uint256 i=0; i<_fragmentNames.length; i++){
            if(keccak256(abi.encodePacked(_fragmentNames[i])) == keccak256(abi.encodePacked(name))){
                nameExists = true;
                break;
            }
        }
        if(!nameExists){
            _fragmentNames.push(name);
        }

        emit FragmentAdded(name, version, msg.sender);
    }
    
    function updateFragment(
        string calldata name,
        uint256 version,
        string calldata newContent
    ) external override onlyOwner {
        SVGFragment storage fragmentEntry = _fragments[name][version];
        require(fragmentEntry.creator != address(0), "Fragment not found");
        require(bytes(newContent).length > 0, "Content cannot be empty");

        fragmentEntry.content = newContent;
        // Assuming a 'lastUpdated' field exists
        emit FragmentUpdated(name, version, msg.sender);
    }

    function getFragment(string calldata name, uint256 version) external view override returns (SVGFragment memory) {
        SVGFragment memory fragmentEntry = _fragments[name][version];
        require(fragmentEntry.creator != address(0), "Fragment not found");
        return fragmentEntry;
    }

    function renderSVG(
        string calldata templateName,
        uint256 templateVersion,
        bytes calldata data
    ) external view override returns (string memory svgURI) {
        SVGTemplate memory template = _templates[templateName][templateVersion];
        require(template.creator != address(0), "Template not found");
        require(template.active == true, "Template inactive");

        string memory svgContent = template.content;

        // Basic example of data integration. A more robust solution would need a templating engine
        // or more structured data parsing. This part highly depends on expected 'data' format.
        // For demonstration, let's assume 'data' is simply a string to be inserted.
        // A more advanced approach would use a dedicated string utility library for replaceAll.
        
        // Example: if data is abi.encode(color, text, number)
        (string memory color, string memory text, uint256 number) = abi.decode(data, (string, string, uint256));
        
        svgContent = _replacePlaceholder(svgContent, "[[COLOR]]", color);
        svgContent = _replacePlaceholder(svgContent, "[[TEXT]]", text);
        svgContent = _replacePlaceholder(svgContent, "[[NUMBER]]", _uint256ToString(number));

        // Example: Inject fragments
        string memory fragment1Content = _fragments["star_icon"][1].content; // Assuming fragment exists
        svgContent = _replacePlaceholder(svgContent, "[[STAR_ICON]]", fragment1Content);

        // Construct data URI
        bytes memory rawSvgBytes = bytes(svgContent);
        string memory base64Encoded = Base64.encode(rawSvgBytes);
        svgURI = string(abi.encodePacked("data:image/svg+xml;base64,", base64Encoded));
        
        // Emit event, potentially storing tokenId and linking SVG for future retrieval
        // emit SVGGenerated(_tokenId, templateName, templateVersion, svgURI);
    }

    function getRenderedSVG(uint256 tokenId) external view override returns (string memory svgURI) {
        // This function would typically retrieve from a mapping if SVGs were stored persistently
        // For this example, assuming it's dynamic or a dummy.
        // If the intention is to store, _renderedSVGStorage[tokenId] would be used.
        return _renderedSVGStorage[tokenId];
    }

    function getTemplateNames() external view override returns (string[] memory) {
        return _templateNames;
    }

    function getAvailableVersions(string calldata templateName) external view override returns (uint256[] memory) {
        return _templateVersions[templateName];
    }
    
    function getFragmentNames() external view override returns (string[] memory) {
        return _fragmentNames;
    }

    // --- Internal String Utility Functions (Simplified for example) ---
    function _replacePlaceholder(string memory source, string memory placeholder, string memory replacement) internal pure returns (string memory) {
        // This is a very basic string replace. For production, consider a robust StringUtils lib.
        bytes memory sourceBytes = bytes(source);
        bytes memory placeholderBytes = bytes(placeholder);
        bytes memory replacementBytes = bytes(replacement);

        uint265 i = 0;
        uint256 j = 0;
        bytes memory result = new bytes(sourceBytes.length); // Max possible size

        while (i < sourceBytes.length) {
            bool found = true;
            if (i + placeholderBytes.length <= sourceBytes.length) {
                for (uint256 p = 0; p < placeholderBytes.length; p++) {
                    if (sourceBytes[i+p] != placeholderBytes[p]) {
                        found = false;
                        break;
                    }
                }
            } else {
                found = false;
            }

            if (found) {
                for (uint256 r = 0; r < replacementBytes.length; r++) {
                    result[j++] = replacementBytes[r];
                }
                i += placeholderBytes.length;
            } else {
                result[j++] = sourceBytes[i++];
            }
        }
        return string(result);
    }
    
    function _uint256ToString(uint256 _i) internal pure returns (string memory _uintAsString) {
        if (_i == 0) {
            return "0";
        }
        uint256 j = _i;
        uint256 len;
        while (j != 0) {
            len++;
            j /= 10;
        }
        bytes memory bstr = new bytes(len);
        uint256 k = len - 1;
        while (_i != 0) {
            bstr[k--] = byte(uint8(48 + _i % 10));
            _i /= 10;
        }
        return string(bstr);
    }
}
```

## Security Considerations

### Access Control
- **`onlyOwner`**: Critical functions for adding, updating, or deactivating SVG templates and fragments must be restricted to the contract owner or an authorized role to prevent malicious or accidental changes to visual assets.

### Gas Costs
- **String Manipulation**: On-chain string concatenation and replacement can be extremely gas-intensive, especially for complex SVGs or frequent rendering. Consider the trade-off between fully on-chain SVG generation and off-chain rendering with on-chain data validation.
- **Base64 Encoding**: `Base64.encode` also consumes significant gas; ensure its usage is justified by the application's needs.

### Input Validation
- **Data `bytes`**: The `data` parameter in `renderSVG` should be carefully parsed and validated to prevent unexpected SVG output or injection vulnerabilities if directly embedded without proper sanitization.
- **Template Content**: Validate that template and fragment content doesn't contain malicious scripts or excessive size.

## Best Practices

### Off-chain Rendering (Alternative)
1. **Hybrid Approach**: For high-volume or very complex SVGs, consider storing only the dynamic numerical/text data on-chain and performing the actual SVG rendering in a frontend application or a dedicated off-chain service. The smart contract would then primarily act as a data oracle.

### Template Design
1. **Placeholders**: Use clear, consistent placeholders (e.g., `[[COLOR]]`, `{{VARIABLE_NAME}}`) within SVG templates to indicate where dynamic data should be injected.
2. **Modular Fragments**: Break down complex SVGs into smaller, reusable fragments to promote efficiency and simplify template management.

### Data URI Length
1. **Transaction Limits**: Be aware of blockchain transaction size limits if you intend to store or return very large SVG data URIs via events or return values.
2. **IPFS/Storage**: For large metadata, store the SVG on IPFS or another decentralized storage solution and just provide the hash/URI on-chain.

## Integration Examples

### Frontend Integration (React with Ethers.js)

```typescript
import React, { useState, useEffect } from 'react';
import { ethers, Contract } from 'ethers';
import SVGABI from './SVGGenerator.json'; // ABI for the ISVG contract

const SVG_GENERATOR_ADDRESS = "0x..."; // Deployed SVG generator contract address

interface SVGComponentProps {
    tokenId: number;
    color: string;
    text: string;
}

const NFTImage: React.FC<SVGComponentProps> = ({ tokenId, color, text }) => {
    const [svgDataUri, setSvgDataUri] = useState<string>('');
    const [loading, setLoading] = useState<boolean>(true);
    const [error, setError] = useState<string | null>(null);

    useEffect(() => {
        const fetchSVG = async () => {
            setLoading(true);
            setError(null);
            try {
                const provider = new ethers.providers.Web3Provider(window.ethereum);
                const svgContract = new Contract(SVG_GENERATOR_ADDRESS, SVGABI, provider);

                // Encode dynamic data
                const encodedData = ethers.utils.defaultAbiCoder.encode(
                    ["string", "string", "uint256"], // Match the abi.decode in contract
                    [color, text, tokenId]
                );

                // Call renderSVG specifying a template and version
                const templateName = "CryptoBadge"; // Example template name
                const templateVersion = 1; // Example template version
                const uri = await svgContract.renderSVG(templateName, templateVersion, encodedData);
                setSvgDataUri(uri);
            } catch (err: any) {
                console.error("Error fetching SVG:", err);
                setError(`Failed to fetch SVG: ${err.message || err}`);
            } finally {
                setLoading(false);
            }
        };

        fetchSVG();
    }, [tokenId, color, text]);

    if (loading) return <div>Loading SVG...</div>;
    if (error) return <div>Error: {error}</div>;

    return (
        <img src={svgDataUri} alt={`NFT ${tokenId}`} style={{ width: '300px', height: '300px' }} />
    );
};

export default NFTImage;
```

### Backend Integration (Node.js/Web3.js for off-chain metadata generation)

```javascript
const Web3 = require('web3');
const SVGGeneratorABI = require('./SVGGenerator.json').abi; // ABI of ISVG contract

const web3 = new Web3('YOUR_ETHEREUM_RPC_URL');
const svgGeneratorAddress = '0x...'; // Deployed SVG generator contract address

const svgGeneratorContract = new web3.eth.Contract(SVGGeneratorABI, svgGeneratorAddress);

async function generateNFTMetadata(tokenId, nftAttributes) {
    try {
        console.log(`Generating SVG for Token ID: ${tokenId}`);

        const templateName = "ArtPieceTemplate";
        const templateVersion = 1; // Assuming version 1 for now

        // ABI encode the attributes
        const encodedData = web3.eth.abi.encodeParameters(
            ['string', 'uint256', 'string'], // Example types: color, rarity, texture
            [nftAttributes.color, nftAttributes.rarity, nftAttributes.texture]
        );

        const svgDataUri = await svgGeneratorContract.methods.renderSVG(
            templateName,
            templateVersion,
            encodedData
        ).call();

        // Construct ERC721 metadata JSON
        const metadata = {
            name: `My NFT #${tokenId}`,
            description: "An algorithmically generated NFT from Gemforce.",
            image: svgDataUri, // The generated SVG data URI
            attributes: [
                { trait_type: "Color", value: nftAttributes.color },
                { trait_type: "Rarity", value: nftAttributes.rarity },
                { trait_type: "Texture", value: nftAttributes.texture }
            ]
        };

        console.log(`Metadata for Token ID ${tokenId}:`, JSON.stringify(metadata, null, 2));
        return metadata;

    } catch (error) {
        console.error('Error generating NFT metadata:', error);
        throw error;
    }
}

// Example Usage
// generateNFTMetadata(123, { color: "blue", rarity: 5, texture: "smooth" });
```

## Related Documentation

- [SVG Templates Library](../libraries/svg-templates-lib.md)
- [ERC721 Metadata Standard](https://eips.ethereum.org/EIPS/eip-721#metadata)
- [Base64 Encoding (OpenZeppelin's Base64)](https://docs.openzeppelin.com/contracts/4.x/api/utils#Base64)
- [Data URIs (MDN Web Docs)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs)

## Standards Compliance

- **ERC721 Metadata**: The output can be directly used as the `image` field for ERC721 token metadata.
- **Ownable**: Utilizes OpenZeppelin's `Ownable` for administrative access control.