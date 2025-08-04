  Automated image processing pipeline orchestrated in Python. This solution
  programmatically transforms a basic product photo into a professional, realistic advertisement by leveraging a stack of open-source AI
  models, ensuring the entire process is self-hostable and free of recurring API costs.

  The technical workflow is executed in four distinct stages:

  Stage 1: Automated Product Segmentation
   * Objective: To isolate the product from its original background with high fidelity.
   * Tech Stack: We use PyTorch to run Meta AI's Segment Anything Model (SAM), specifically the vit_l checkpoint.
   * Process: The user's image is loaded, and we automate the segmentation by providing SAM's SamPredictor with a programmatic bounding box
     that encompasses 95% of the image canvas. This bypasses the need for manual input. The resulting binary mask is then applied as an
     alpha channel to the original image using OpenCV, and the final RGBA asset is cropped to its content boundaries with cv2.boundingRect.

  Stage 2: Generative Background Creation
   * Objective: To generate a new, artistic background scene from a text prompt.
   * Tech Stack: A self-hosted Stable Diffusion instance is accessed via the AUTOMATIC1111 API.
   * Process: We send a POST request to the /sdapi/v1/txt2img endpoint with a JSON payload containing the user's prompt and generation
     parameters. The returned base64 image is decoded into a NumPy array.

  Stage 3: Enhanced Compositing with Realism Features
   * Objective: To place the product onto the background realistically, avoiding distortion and adding depth.
   * Tech Stack: This stage relies heavily on OpenCV and Pillow (PIL) for precise image manipulation.
   * Process:
       1. Aspect-Ratio-Preserving Resize: To prevent distortion, we calculate a new size for the product that fits within a predefined bounding
          box (e.g., 70% of the background's width) while maintaining its original aspect ratio.
       2. Subtle Color Matching: We use cv2.addWeighted to apply a slight color tint to the product, helping it match the color temperature of
          the generated background.
       3. Soft Drop Shadow: A gray silhouette is created from the product's alpha mask. This shape is heavily blurred using cv2.GaussianBlur
          with a large kernel (e.g., (75, 75)) to create a soft, diffuse shadow. This shadow is then blended onto the background at a low
          opacity (e.g., 30%) before the product is placed on top, adding a crucial layer of depth.

  Stage 4: Contextual Blending via Inpainting
   * Objective: To seamlessly integrate the product by generating context-aware lighting and contact shadows.
   * Tech Stack: We use the img2img functionality of the Stable Diffusion API.
   * Process:
       1. A new mask is created that corresponds to the exact location of the composited product.
       2. We make a POST request to the /sdapi/v1/img2img endpoint, providing the composited image and the product mask.
       3. The critical parameter is inpainting_mask_invert: 1. This inverts the mask, instructing Stable Diffusion to lock the product's pixels
          and only regenerate the background area immediately surrounding it.
       4. By setting a moderate denoising_strength (e.g., 0.35), the model subtly alters the background to create realistic lighting,
          reflections, and contact shadows that conform perfectly to the product's shape, effectively "grounding" it in the new environment for
          a photo-realistic finish.
