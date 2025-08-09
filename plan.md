```markdown
# Detailed Implementation Plan for Lok Sewa Aayog MCQ Creator

Below is the step-by-step plan covering all dependent files and changes.

---

## 1. Define Data Types and Schemas

**File:** `src/types/question.ts`  
- Create an interface for a multiple choice question.  
- Define the properties:  
  - id (string, generated using UUID)  
  - questionText (string)  
  - options (array of strings – ideally 4 options)  
  - correctAnswerIndex (number, index of the correct option)  
- Create a Zod schema (e.g., `questionSchema`) for validation in both the client and API.  
- Use template literals to build any string constants if necessary.

**Example Code:**
```typescript
import { z } from "zod";

export interface Question {
  id: string;
  questionText: string;
  options: string[];
  correctAnswerIndex: number;
}

export const questionSchema = z.object({
  id: z.string(),
  questionText: z.string().min(1, "Question text cannot be empty"),
  options: z.array(z.string().min(1, "Option cannot be empty")).length(4, "Exactly four options required"),
  correctAnswerIndex: z.number().int().min(0).max(3),
});
```

---

## 2. Build the API Endpoint for Question Submission

**File:** `src/app/api/questions/route.ts`  
- Create a new API route that handles POST requests.  
- Import the question schema from the types file using the correct relative path (`../../../types/question`).  
- Use a try-catch block to validate incoming JSON data with the Zod schema.  
- On validation failure, return a 400 response with error details.  
- Simulate saving the question (e.g., return the received data with a generated id) and return HTTP 201.

**Example Code:**
```typescript
import { NextResponse } from "next/server";
import { questionSchema } from "../../../types/question";

export async function POST(request: Request) {
  try {
    const jsonData = await request.json();
    // Validate incoming data
    const parsed = questionSchema.parse(jsonData);
    // Here, you might save the data to a database. For now, echo back.
    return NextResponse.json({ message: "Question created successfully", question: parsed }, { status: 201 });
  } catch (error: any) {
    return NextResponse.json({ error: error.errors ?? error.message }, { status: 400 });
  }
}
```

---

## 3. Create the Question Creation UI Page

**File:** `src/app/create-question/page.tsx`  
- Mark the component as a client component by adding `"use client"` at the top.  
- Use React, react-hook-form, and Zod for client-side form handling and validation.  
- Build a modern form UI using Shadcn/ui components (e.g., Card, Input, Label, Button) styled with Tailwind CSS.  
- The form should include:  
  - A textarea for the question text  
  - Four input fields for the options  
  - A radio group to select the correct answer (e.g., one radio button per option)  
  - A submit button with loading and error states  
- On form submission, post data to `/api/questions` using fetch.  
- Handle success (e.g., clear the form and show a success message) and error display.

**Example Code:**
```tsx
"use client";

import React from "react";
import { useForm, SubmitHandler, Controller } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { questionSchema, Question } from "@/types/question";
import { z } from "zod";

const defaultValues: Omit<Question, "id"> = {
  questionText: "",
  options: ["", "", "", ""],
  correctAnswerIndex: 0,
};

export default function CreateQuestionPage() {
  const {
    register,
    handleSubmit,
    control,
    reset,
    formState: { errors, isSubmitting },
  } = useForm<Omit<Question, "id">>({
    resolver: zodResolver(questionSchema.omit({ id: true })),
    defaultValues,
  });

  const onSubmit: SubmitHandler<Omit<Question, "id">> = async (data) => {
    try {
      const response = await fetch("/api/questions", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ ...data, id: new Date().getTime().toString() }),
      });
      if (!response.ok) {
        const errData = await response.json();
        alert("Error: " + JSON.stringify(errData.error));
        return;
      }
      const result = await response.json();
      alert(result.message);
      reset();
    } catch (error) {
      alert("Network error, please try again.");
    }
  };

  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-2xl font-semibold mb-4">Create MCQ for Lok Sewa Aayog</h1>
      <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
        <div>
          <label className="block font-medium mb-1">Question Text</label>
          <textarea
            {...register("questionText")}
            className="w-full p-2 border rounded"
            placeholder="Enter your question here"
          />
          {errors.questionText && <p className="text-red-600">{errors.questionText.message}</p>}
        </div>
        {defaultValues.options.map((_, index) => (
          <div key={index} className="space-y-1">
            <label className="block font-medium">Option {index + 1}</label>
            <input
              type="text"
              {...register(`options.${index}`)}
              className="w-full p-2 border rounded"
              placeholder={`Enter option ${index + 1}`}
            />
            {errors.options && errors.options[index] && (
              <p className="text-red-600">{errors.options[index]?.message}</p>
            )}
          </div>
        ))}
        <div>
          <label className="block font-medium mb-1">Select Correct Option:</label>
          <div className="flex space-x-4">
            {defaultValues.options.map((_, index) => (
              <label key={index} className="flex items-center">
                <input
                  type="radio"
                  value={index}
                  {...register("correctAnswerIndex")}
                  className="mr-1"
                  defaultChecked={index === 0}
                />
                Option {index + 1}
              </label>
            ))}
          </div>
          {errors.correctAnswerIndex && (
            <p className="text-red-600">{errors.correctAnswerIndex.message}</p>
          )}
        </div>
        <button type="submit" disabled={isSubmitting} className="px-4 py-2 bg-primary text-primary-foreground rounded">
          {isSubmitting ? "Submitting..." : "Create Question"}
        </button>
      </form>
    </div>
  );
}
```

---

## 4. UI/UX and Integration Considerations

- **Design:**  
  - Use consistent typography, spacing, and color variables defined in `globals.css`.  
  - The form should be clean and modern, without external icons or images, respecting the design guidelines.

- **Navigation:**  
  - Optionally, add a link to the new “Create Question” page in an existing navigation component (if applicable) so administrators can access it easily.

- **Error Handling:**  
  - Client-side errors (form validation) are shown next to each field.  
  - API errors are caught and displayed using alert dialogs.  
  - Network errors also trigger clear feedback.

- **Testing:**  
  - Validate the API endpoint using curl:
    ```bash
    curl -X POST http://localhost:8000/api/questions \
    -H "Content-Type: application/json" \
    -d '{"id": "test", "questionText": "Sample Question?", "options": ["A", "B", "C", "D"], "correctAnswerIndex": 2}'
    ```
  - Ensure form submission and UI responses work as expected.

---

## Summary

- Created a new type definition file in `src/types/question.ts` with an interface and Zod schema.
- Developed an API endpoint in `src/app/api/questions/route.ts` to validate and handle POST requests.
- Built a client-side page in `src/app/create-question/page.tsx` using react-hook-form and TailwindCSS for a modern UI.
- Integrated error handling both on the client and server sides.
- Ensured consistent UI/UX by leveraging existing Tailwind variables and Shadcn/ui components.
- Included testing instructions via curl to validate the API functionality.
