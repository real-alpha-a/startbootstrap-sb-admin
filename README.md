import os
import logging
from typing import Dict, Any

def load_and_format_sql(sql_path: str, params: Dict[str, Any]) -> str:
    """
    Reads a SQL template file and formats it with the provided parameters.
    
    Args:
        sql_path (str): Path to the .sql template.
        params (dict): Dictionary containing keys matching the SQL placeholders 
                       (e.g., {'project': '...', 'dataset': '...'}).
                       
    Returns:
        str: The ready-to-execute SQL query.
    """
    if not os.path.exists(sql_path):
        raise FileNotFoundError(f"SQL template not found: {sql_path}")
        
    try:
        with open(sql_path, 'r') as f:
            raw_query = f.read()
            
        # strict=False allows extra keys in 'params' without crashing
        # strict=True (default behavior) creates errors if SQL keys are missing in params
        formatted_query = raw_query.format(**params)
        
        return formatted_query.strip()
        
    except KeyError as e:
        logging.error(f"Missing parameter for SQL template: {e}")
        raise ValueError(f"Template requires missing key: {e}")
    except Exception as e:
        logging.error(f"Error formatting SQL: {e}")
        raise
