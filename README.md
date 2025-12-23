# Ballerina Gemini Connector

The Ballerina Gemini Connector provides a seamless interface to interact with the Google Gemini API, enabling developers to integrate advanced generative AI capabilities into their Ballerina applications.

## Features

The connector provides the following features:

- **[Generate Content](#generate-content):** Generate text responses using Gemini models.
- **[Structured Output](#structured-output-json):** Generate JSON responses with defined schemas.
- **[Function Calling](#function-calling):** Connect models to external tools and APIs.
- **[Image Understanding](#image-understanding):** Process and analyze images (multimodal capabilities).
- **[Image Generation](#image-generation):** Generate images using Gemini models.
- **[Document Processing](#document-processing):** Process and analyze PDF documents.
- **[Grounding with Google Search](#grounding-with-google-search):** Access real-time data with citations.
- **[Code Execution](#code-execution):** Generate and run Python code for complex tasks.
- **[Grounding with Google Maps](#grounding-with-google-maps):** Ground responses in real-world data using Google Maps.
- **[Interactions API](#interactions-api):** Multi-turn conversations and stateful interactions with models and agents.
- **[File Search](#file-search):** Retrieve relevant information from documents to answer questions (Retrieval Augmented Generation).
- **[URL Context](#url-context):** Provide context via URLs for the model to access.


# Quickstart Guide

## Generate Content

To use the Gemini connector in your Ballerina application, update the `.bal` file as follows:

### Step 1: Import the module
Import the `qaadir/gemini` module.

```ballerina
import qaadir/gemini;
import ballerina/io;
```

### Step 2: Create a new connector instance
Create a `gemini:Client` with the obtained API Key and initialize the connector.

```ballerina
configurable string apiKey = ?;

final gemini:Client geminiClient = check new ({
    xGoogAPIKey: apiKey
});
```

### Step 3: Invoke the connector operation
Now, you can utilize available connector operations.

**Generate a response for a given message**

```ballerina
public function main() returns error? {
    
    // Create a content generation request.
    gemini:GenerateContentRequest request = {
        contents: [
            {
                role: "user",
                parts: [{text: "Explain quantum computing in simple terms."}]
            }
        ]
    };

    // Invoke the generateContent operation.
    // The model name (e.g., "gemini-2.5-flash") is passed as a path parameter.
    gemini:GenerateContentResponse response = check geminiClient->generativeServiceGenerateContent("gemini-2.5-flash", request);

    // Process the response
    if response.candidates is gemini:Candidate[] && response.candidates.length() > 0 {
         io:println(response.candidates[0].content?.parts[0].text);
    }
}
```

### Step 4: Run the Ballerina application

```bash
bal run
```

## Structured Output (JSON)

You can force the model to respond with a specific JSON structure by defining a schema in the request configuration.

### Step 1: Define the Schema
Define the structure you want the model to return using the `gemini:Schema` type.

```ballerina
// Define key properties for the schema
gemini:Schema recipeSchema = {
    'type: "OBJECT",
    properties: {
        "recipe_name": { 'type: "STRING", description: "Name of the recipe" },
        "ingredients": {
            'type: "ARRAY",
            items: {
                'type: "OBJECT",
                properties: {
                    "name": { 'type: "STRING" },
                    "quantity": { 'type: "STRING" }
                },
                required: ["name", "quantity"]
            }
        }
    },
    required: ["recipe_name", "ingredients"]
};
```

### Step 2: Configure the Request
Create the request, specifying the `generationConfig` with `responseMimeType` set to `application/json` and your schema.

```ballerina
gemini:GenerateContentRequest request = {
    contents: [
        {
            role: "user",
            parts: [{text: "Extract recipe from: Mix 1 cup flour and 2 eggs..."}]
        }
    ],
    generationConfig: {
        responseMimeType: "application/json",
        responseSchema: recipeSchema
    }
};
```

### Step 3: Invoke the API
Call the `generativeServiceGenerateContent` function. The response text will be a valid JSON string adhering to your schema.

```ballerina
gemini:GenerateContentResponse response = check geminiClient->generativeServiceGenerateContent("gemini-2.5-flash", request);

// The 'text' part of the response will contain the JSON string
gemini:Candidate[]? candidates = response.candidates;
if candidates is gemini:Candidate[] && candidates.length() > 0 {
    io:println(candidates[0].content?.parts[0].text);
}
```

## Function Calling

Function calling lets you connect models to external tools and APIs. You define the function signature, and the model can decide to call it to fulfill a user's request.

### Step 1: Define the Function Declaration
Define the metadata of the function (name, description, parameters) you want to expose to the model.

```ballerina
gemini:FunctionDeclaration scheduleMeetingFunction = {
    name: "schedule_meeting",
    description: "Schedules a meeting with specified attendees at a given time and date.",
    parameters: {
        'type: "OBJECT",
        properties: {
            "attendees": {
                'type: "ARRAY",
                items: {'type: "STRING"},
                description: "List of people attending the meeting."
            },
            "date": {
                'type: "STRING",
                description: "Date of the meeting (e.g., '2024-07-29')"
            },
            "time": {
                'type: "STRING",
                description: "Time of the meeting (e.g., '15:00')"
            },
            "topic": {
                'type: "STRING",
                description: "The subject or topic of the meeting."
            }
        },
        required: ["attendees", "date", "time", "topic"]
    }
};
```

### Step 2: Wrap it in a Tool
The API expects a `Tool` object, which can contain a list of function declarations.

```ballerina
gemini:Tool meetingTool = {
    functionDeclarations: [scheduleMeetingFunction]
};
```

### Step 3: Configure the Request
Create the request and pass the `Tool` object in the `tools` field.

```ballerina
gemini:GenerateContentRequest request = {
    contents: [
        {
            role: "user",
            parts: [{text: "Schedule a meeting with Bob and Alice for 03/14/2025 at 10:00 AM about the Q3 planning."}]
        }
    ],
    tools: [meetingTool]
};
```

### Step 4: Invoke the API
Call the content generation endpoint.

```ballerina
gemini:GenerateContentResponse response = check geminiClient->generativeServiceGenerateContent("gemini-2.5-flash", request);
```

### Step 5: Handle the Function Call
Check if the response contains a `functionCall`. If so, extract the name and arguments to execute your local logic.

```ballerina
gemini:Candidate[]? candidates = response.candidates;
if candidates is gemini:Candidate[] && candidates.length() > 0 {
    gemini:Part[]? parts = candidates[0].content?.parts;
    if parts is gemini:Part[] && parts.length() > 0 {
        gemini:FunctionCall? functionCall = parts[0].functionCall;
        
        if functionCall is gemini:FunctionCall {
             io:println("Function to call: ", functionCall.name);
             io:println("Arguments: ", functionCall.args.toString());
             // Execute your local function logic using arguments
        }
    }
}
```

## Image Understanding

Gemini models are multimodal and can process images. You can pass image data inline (Base64 encoded) or via file URIs.

### Step 1: Read and Encode the Image
Read the image file as bytes and encode it to a Base64 string.

```ballerina
import ballerina/io;
import ballerina/lang.array;

string imagePath = "path/to/image.png";
byte[] imageBytes = check io:fileReadBytes(imagePath);
string base64Image = array:toBase64(imageBytes);
```

### Step 2: Configure the Request
Create the request using `inlineData` within a `Part`.

```ballerina
gemini:GenerateContentRequest request = {
    contents: [
        {
            role: "user",
            parts: [
                {
                    inlineData: {
                        mimeType: "image/png", 
                        data: base64Image
                    }
                },
                {
                    text: "Describe this image in detail."
                }
            ]
        }
    ]
};
```

### Step 3: Invoke the API and Handle Response
Call the API and print the generated text description.

```ballerina
gemini:GenerateContentResponse response = check geminiClient->generativeServiceGenerateContent("gemini-2.5-flash", request);

gemini:Candidate[]? candidates = response.candidates;
if candidates is gemini:Candidate[] && candidates.length() > 0 {
    gemini:Content? content = candidates[0].content;
    if content is gemini:Content {
        gemini:Part[]? parts = content.parts;
        if parts is gemini:Part[] && parts.length() > 0 {
            io:println("Response: ", parts[0].text);
        }
    }
}
```

## Image Generation

Gemini can generate images from text prompts. You can request image generation by specifying the response modality.

### Step 1: Configure the Request
Use an image-generation capable model (e.g., `gemini-2.5-flash-image`) and set `responseModalities` to request images.

> **Note:** The integer `2` corresponds to the `IMAGE` modality in the API.

```ballerina
gemini:GenerateContentRequest request = {
    contents: [
        {
            role: "user",
            parts: [{text: "A futuristic city with flying cars, neon lights, and a giant holographic billboard."}]
        }
    ],
    generationConfig: {
        responseModalities: [2], // Request IMAGE output
        responseMimeType: "image/png"
    }
};
```

### Step 2: Invoke the API
Call the API using the appropriate model.

```ballerina
gemini:GenerateContentResponse response = check geminiClient->generativeServiceGenerateContent("gemini-2.5-flash-image", request);
```

### Step 3: Handle the Response
The generated image will be returned as inline data (Base64 encoded). You need to decode it and save it to a file.

```ballerina
import ballerina/io;
import ballerina/lang.array;

gemini:Candidate[]? candidates = response.candidates;
if candidates is gemini:Candidate[] && candidates.length() > 0 {
    gemini:Content? content = candidates[0].content;
    if content is gemini:Content {
        gemini:Part[]? parts = content.parts;
        if parts is gemini:Part[] {
            foreach gemini:Part part in parts {
                // Check for inline image data
                if part.inlineData is gemini:Blob {
                    gemini:Blob blob = <gemini:Blob>part.inlineData;
                    string? base64Data = blob.data;
                    if base64Data is string {
                        // Decode and save the image
                        byte[] imageBytes = check array:fromBase64(base64Data);
                        check io:fileWriteBytes("./generated-image.png", imageBytes);
                        io:println("Image saved to ./generated-image.png");
                    }
                } 
            }
        }
    }
}
```

## Document Processing

Gemini can process and analyze PDF documents.

### Step 1: Read and Encode the Document
Read the PDF file as bytes and encode it to a Base64 string.

```ballerina
import ballerina/io;
import ballerina/lang.array;

string pdfPath = "path/to/document.pdf";
byte[] pdfBytes = check io:fileReadBytes(pdfPath);
string base64Pdf = array:toBase64(pdfBytes);
```

### Step 2: Configure the Request
Create the request using `inlineData` with MIME type `application/pdf`.

```ballerina
gemini:GenerateContentRequest request = {
    contents: [
        {
            role: "user",
            parts: [
                {
                    inlineData: {
                        mimeType: "application/pdf", 
                        data: base64Pdf
                    }
                },
                {
                    text: "Summarize this document."
                }
            ]
        }
    ]
};
```

### Step 3: Invoke the API
Call the model (e.g., `gemini-2.5-flash`) to get the summary.

```ballerina
gemini:GenerateContentResponse response = check geminiClient->generativeServiceGenerateContent("gemini-2.5-flash", request);

gemini:Candidate[]? candidates = response.candidates;
if candidates is gemini:Candidate[] && candidates.length() > 0 {
    io:println("Summary: ", candidates[0].content?.parts[0].text);
}
```

## Grounding with Google Search

Ground your model's responses in real-world data using Google Search.

### Step 1: Define the Search Tool
Enable the Google Search tool.

```ballerina
gemini:Tool searchTool = {
    googleSearch: {}
};
```

### Step 2: Configure the Request
Add the tool to your request configuration.

```ballerina
gemini:GenerateContentRequest request = {
    contents: [
        {
            role: "user",
            parts: [{text: "What happened at the last Olympics?"}]
        }
    ],
    tools: [searchTool]
};
```

### Step 3: Invoke the API and Print Citations
The response will include the answer and grounding metadata (citations).

```ballerina
gemini:GenerateContentResponse response = check geminiClient->generativeServiceGenerateContent("gemini-2.5-flash", request);

gemini:Candidate[]? candidates = response.candidates;
if candidates is gemini:Candidate[] && candidates.length() > 0 {
    // Print text response
    io:println("Response: ", candidates[0].content?.parts[0].text);
    
    // Process citations
    gemini:GroundingMetadata? metadata = candidates[0].groundingMetadata;
    if metadata is gemini:GroundingMetadata {
         gemini:GroundingChunk[]? chunks = metadata.groundingChunks;
         if chunks is gemini:GroundingChunk[] {
             io:println("--- Sources ---");
             foreach gemini:GroundingChunk chunk in chunks {
                 io:println(chunk.web?.uri);
             }
         }
    }
}
```

## Code Execution

The model can generate and execute Python code to answer complex questions (e.g., math, data analysis).

### Step 1: Define the Code Execution Tool
Enable the Code Execution tool.

```ballerina
gemini:Tool codeTool = {
    codeExecution: {}
};
```

### Step 2: Configure the Request
Pass the tool in your request.

```ballerina
gemini:GenerateContentRequest request = {
    contents: [
        {
            role: "user",
            parts: [{text: "Sum the first 50 prime numbers."}]
        }
    ],
    tools: [codeTool]
};
```

### Step 3: Invoke the API and Handle Execution Results
The response contains both the generated code and the execution result.

```ballerina
gemini:GenerateContentResponse response = check geminiClient->generativeServiceGenerateContent("gemini-2.5-flash", request);

gemini:Candidate[]? candidates = response.candidates;
if candidates is gemini:Candidate[] && candidates.length() > 0 {
    gemini:Part[]? parts = candidates[0].content?.parts;
    if parts is gemini:Part[] {
        foreach gemini:Part part in parts {
             // Print generated code
             if part.executableCode is gemini:ExecutableCode {
                 gemini:ExecutableCode exeCode = <gemini:ExecutableCode>part.executableCode;
                 io:println("Generated Code: ", exeCode.code);
             }
             // Print execution result
             if part.codeExecutionResult is gemini:CodeExecutionResult {
                 gemini:CodeExecutionResult execResult = <gemini:CodeExecutionResult>part.codeExecutionResult;
                 io:println("Result: ", execResult.output);
             }
        }
    }
}
```

## URL Context

Provide URLs as context for the model to analyze.

### Step 1: Define the URL Context Tool
Enable the URL Context tool.

```ballerina
gemini:Tool urlTool = {
    urlContext: {}
};
```

### Step 2: Configure the Request
Include the tool and mention URLs in your prompt.

```ballerina
gemini:GenerateContentRequest request = {
    contents: [
        {
            role: "user",
            parts: [{text: "Summarize: https://example.com"}]
        }
    ],
    tools: [urlTool]
};
```

### Step 3: Invoke and Verification
The response includes metadata about the URL retrieval status.

```ballerina
gemini:GenerateContentResponse response = check geminiClient->generativeServiceGenerateContent("gemini-2.5-flash", request);

gemini:Candidate[]? candidates = response.candidates;
if candidates is gemini:Candidate[] && candidates.length() > 0 {
    // Print response text
    gemini:Part[]? parts = candidates[0].content?.parts;
    if parts is gemini:Part[] && parts.length() > 0 {
        io:println("Response: ", parts[0].text);
    }
    
    // Check URL metadata
    gemini:UrlContextMetadata? metadata = candidates[0].urlContextMetadata;
    if metadata is gemini:UrlContextMetadata {
        foreach gemini:UrlMetadata urlData in metadata.urlMetadata ?: [] {
            io:println("URL Status: ", urlData.urlRetrievalStatus);
        }
    }
}
```

## Grounding with Google Maps

Ground your model's responses in real-world data using Google Maps (e.g., finding places near a location).

### Step 1: Define the Google Maps Tool
Enable the Google Maps tool and optionally request the widget token.

```ballerina
gemini:Tool mapsTool = {
    googleMaps: {
        enableWidget: true
    }
};
```

### Step 2: Define Location Context
Specify the location (latitude/longitude) to ground the request in (e.g., Los Angeles).

```ballerina
gemini:RetrievalConfig retrievalConfig = {
    latLng: {
        latitude: 34.050481,
        longitude: -118.248526
    }
};

gemini:ToolConfig toolConfig = {
    retrievalConfig: retrievalConfig
};
```

### Step 3: Configure the Request
Include the tool and the tool configuration in your request.

```ballerina
gemini:GenerateContentRequest request = {
    contents: [
        {
            role: "user",
            parts: [{text: "What are the best Italian restaurants within a 15-minute walk from here?"}]
        }
    ],
    tools: [mapsTool],
    toolConfig: toolConfig
};
```

### Step 4: Invoke and Verification
The response includes the answer, citations (sources), and potentially a widget token.

```ballerina
gemini:GenerateContentResponse response = check geminiClient->generativeServiceGenerateContent("gemini-2.5-flash", request);

gemini:Candidate[]? candidates = response.candidates;
if candidates is gemini:Candidate[] && candidates.length() > 0 {
    // Print response text
    gemini:Part[]? parts = candidates[0].content?.parts;
    if parts is gemini:Part[] && parts.length() > 0 {
        io:println("Response: ", parts[0].text);
    }

    // Process Grounding Metadata
    gemini:GroundingMetadata? metadata = candidates[0].groundingMetadata;
    if metadata is gemini:GroundingMetadata {
        // Print Sources (Grounding Chunks)
        foreach gemini:GroundingChunk chunk in metadata.groundingChunks ?: [] {
             if chunk.maps is gemini:GroundingChunk_Maps {
                 io:println("Source: ", chunk.maps.title, " - ", chunk.maps.uri);
             }
        }
        
        // Print Widget Token
        if metadata.googleMapsWidgetContextToken is string {
            io:println("Widget Token: ", metadata.googleMapsWidgetContextToken);
        }
    }
}
```

## Interactions API

The Interactions API allows for multi-turn conversations and stateful interactions with models and agents.

### Step 1: Initialize the Client
Initialize the `gemini:Client` with your API key.

```ballerina
configurable string geminiAPIKey = ?;

gemini:Client geminiClient = check new ({
    xGoogAPIKey: geminiAPIKey
});
```

### Step 2: Create the Interaction Request
Define the `CreateInteractionRequest` with the model and input.

```ballerina
gemini:CreateInteractionRequest request = {
    model: "gemini-3-flash-preview", 
    input: "Tell me a short joke about programming."
};
```

### Step 3: Invoke the Interactions API
Call the `interactionsCreate` method on the client.

```ballerina
gemini:Interaction|error response = geminiClient->interactionsCreate(request);
```

### Step 4: Handle the Response
Process the response to retrieve the generated content.

```ballerina
if response is gemini:Interaction {
    io:println("Interaction Created Successfully!");
    io:println("Interaction ID: ", response.id);
    
    // Print the output text
    gemini:Content[]? outputs = response.outputs;
    if outputs is gemini:Content[] && outputs.length() > 0 {
         foreach var output in outputs {
             foreach var part in output.parts ?: [] {
                 if part.text != () {
                     io:println("Model Response: ", part.text);
                 }
             }
         }
    }
} else {
    io:println("Error creating interaction: ", response);
}
```

## File Search (RAG)

File Search allows you to create a store of documents (PDFs, text files, etc.) and have the model retrieve relevant information from them to answer questions (Retrieval Augmented Generation).

### Step 1: Upload a File
First, upload the file you want to search.

```ballerina
import ballerina/io;

// Read file content
 byte[] fileBytes = check io:fileReadBytes("policy.pdf");

// Upload file (returns a FileUploadResponse wrapping a FileResource)
gemini:FileUploadResponse uploadResp = check geminiClient->filesUploadBytes(fileBytes, "application/pdf", "policy.pdf");
gemini:FileResource fileResource = uploadResp.file;
io:println("File URI: ", fileResource.uri);
```

### Step 2: Create a File Search Store
Create a logical store to hold your files.

```ballerina
gemini:CreateFileSearchStoreRequest storeRequest = {
    fileSearchStore: {
        displayName: "Company Policies"
    }
};

gemini:FileSearchStore store = check geminiClient->fileSearchStoresCreate(storeRequest);
string storeName = store.name ?: "";
```

### Step 3: Import File into Store
Link the uploaded file to the store. This starts the ingestion process.

```ballerina
gemini:ImportFileRequest importRequest = {
    fileName: fileResource.name ?: ""
};

// Returns an Operation (LRO)
gemini:ImportFileOperation operation = check geminiClient->fileSearchStoresImportFile(storeName, importRequest);

// Wait for ingestion (simplified for example; in production, consider polling)
runtime:sleep(10.0); 
```

### Step 4: Query with File Search Tool
Generate content using the `fileSearch` tool pointing to your store.

```ballerina
// Define the File Search tool
gemini:ToolWithFileSearch fileSearchTool = {
    fileSearch: {
        fileSearchStoreNames: [storeName]
    }
};

// Create the request
gemini:GenerateContentRequestWithFileSearch genRequest = {
    model: "gemini-1.5-pro",
    contents: [{
        parts: [{text: "What is the policy on remote work?"}]
    }],
    tools: [fileSearchTool]
};

// Invoke the API
gemini:GenerateContentResponse response = check geminiClient->generateContentWithFileSearch("gemini-1.5-pro", genRequest);

// Correctly handle the response
if response.candidates is gemini:Candidate[] && response.candidates.length() > 0 {
     io:println("Answer: ", response.candidates[0].content?.parts[0].text);
}
```
