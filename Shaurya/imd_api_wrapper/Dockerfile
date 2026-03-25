FROM python:3.10-slim
WORKDIR /app

# UPDATE THIS: Tell it to look in the subfolder
COPY imd_api_wrapper/requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

# UPDATE THIS: Copy the rest of the API code from that folder
COPY imd_api_wrapper/ .

EXPOSE 8000

# Ensure this points to the correct location of your app inside the container
CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]