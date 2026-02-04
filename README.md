from google.cloud import storage
from google.api_core.exceptions import NotFound, Forbidden, GoogleAPICallError
from pathlib import Path
import os

def download_gcs_file_safe(gcs_url: str, local_dir: str = "/tmp") -> str:
    """
    Downloads a file from GCS and returns the local file path.
    Raises meaningful exceptions for common failure scenarios.
    """
    if not gcs_url.startswith("gs://"):
        raise ValueError("Invalid GCS URL. Must start with 'gs://'")

    try:
        _, path = gcs_url.split("gs://", 1)
        bucket_name, blob_name = path.split("/", 1)

        client = storage.Client()
        bucket = client.bucket(bucket_name)

        # Check bucket exists
        if not bucket.exists():
            raise FileNotFoundError(f"GCS bucket not found: {bucket_name}")

        blob = bucket.blob(blob_name)

        # Check file exists
        if not blob.exists():
            raise FileNotFoundError(f"GCS file not found: gs://{bucket_name}/{blob_name}")

        Path(local_dir).mkdir(parents=True, exist_ok=True)
        local_path = os.path.join(local_dir, os.path.basename(blob_name))

        blob.download_to_filename(local_path)
        return local_path

    except Forbidden:
        raise PermissionError(f"Access denied to GCS bucket or file: {gcs_url}")

    except NotFound:
        raise FileNotFoundError(f"GCS resource not found: {gcs_url}")

    except GoogleAPICallError as e:
        raise RuntimeError(f"GCS API error while accessing {gcs_url}: {e}")

    except Exception as e:
        raise RuntimeError(f"Unexpected error while downloading {gcs_url}: {e}")
