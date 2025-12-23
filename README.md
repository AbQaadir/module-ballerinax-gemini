# Ballerina Gemini Connector

The Ballerina Gemini Connector provides a seamless interface to interact with the Google Gemini API, enabling developers to integrate advanced generative AI capabilities into their Ballerina applications.

## Features

The connector provides the following features:

- **[Generate Content](#generate-content):** Generate text responses using Gemini models.
- **[Structured Output](#structured-output-json):** Generate JSON responses with defined schemas.
- **[Function Calling](#function-calling):** Connect models to external tools and APIs.
- **[Image Understanding](#image-understanding):** Process and analyze images (multimodal capabilities).
- **[Image Generation](#image-generation):** Generate images using Gemini models.


# Quickstart Guide

## Generate Content

To use the Gemini connector in your Ballerina application, update the `.bal` file as follows:

### Step 1: Import the module
Import the `qaadir/gemini` module.

```ballerina
import qaadir/gemini;
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
    if response.candidates is gemini:Candidate[] {
         // Handle candidates...
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
    gemini:Part[]? parts = candidates[0].content.parts;
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

