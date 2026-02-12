Creates dense point samples inside target polygon feature classes (e.g., mineral polygons).
The script scans polygon FCs with a given prefix (default: 'o'), then generates a dense point grid
inside each polygon using one of three strategies depending on polygon area:
  1) Very small polygons: centroid only
  2) Small polygons: simple grid (spacing-based)
  3) Medium polygons: fishnet-based sampling

Requirements:
- ArcGIS Pro Python environment with arcpy available.
- Polygon feature classes stored in the workspace GDB.

from __future__ import annotations

import os
import math
import logging
from dataclasses import dataclass
from typing import List, Optional

import arcpy


# ================================
# Section 1: Logging
# ================================

def configure_logging(level: int = logging.INFO) -> None:
    """Configure console logging."""
    logging.basicConfig(
        level=level,
        format="%(asctime)s | %(levelname)s | %(message)s",
    )


# ================================
# Section 2: Configuration
# ================================

@dataclass(frozen=True)
class Config:
    """Runtime configuration."""
    workspace_gdb: str
    polygon_prefix: str = "o"
    overwrite_output: bool = True

    # Strategy thresholds (area in square meters)
    centroid_area_threshold: float = 10.0
    grid_area_threshold: float = 100.0

    # Grid strategy spacing (meters) for small polygons
    simple_grid_spacing_m: float = 0.25

    # Fishnet constraints (meters)
    fishnet_cell_min_m: float = 0.5
    fishnet_cell_max_m: float = 5.0


# ================================
# Section 3: Utilities
# ================================

def safe_delete(dataset: str) -> None:
    """Delete a dataset if it exists."""
    try:
        if arcpy.Exists(dataset):
            arcpy.Delete_management(dataset)
    except Exception:
        # Best-effort cleanup; do not raise.
        pass


def ensure_overwrite(overwrite: bool) -> None:
    """Set ArcPy overwrite behavior."""
    arcpy.env.overwriteOutput = bool(overwrite)


def set_workspace(workspace_gdb: str) -> None:
    """Set ArcPy workspace."""
    arcpy.env.workspace = workspace_gdb


def list_target_polygons(prefix: str) -> List[str]:
    """
    List polygon feature classes starting with a prefix.

    Returns:
        List of feature class names (not full paths) in the current workspace.
    """
    targets: List[str] = []
    for fc in (arcpy.ListFeatureClasses() or []):
        if not fc.startswith(prefix):
            continue
        try:
            desc = arcpy.Describe(fc)
            if getattr(desc, "shapeType", "").lower() == "polygon":
                targets.append(fc)
        except Exception:
            continue
    return targets


def polygon_metrics(polygon_fc: str) -> tuple[float, float, float]:
    """
    Approximate polygon metrics from extent.

    Returns:
        (width_m, height_m, area_m2) computed from the feature extent.
    """
    desc = arcpy.Describe(polygon_fc)
    extent = desc.extent
    width = float(extent.XMax - extent.XMin)
    height = float(extent.YMax - extent.YMin)
    area = width * height
    return width, height, area


# ================================
# Section 4: Point Generation Methods
# ================================

def create_dense_points_in_small_polygon(
    polygon_fc: str,
    output_fc: str,
    points_per_meter: float = 0.5,
) -> int:
    """
    Create a dense regular grid of points for a small polygon, then keep only points within polygon.

    Args:
        polygon_fc: Input polygon feature class.
        output_fc: Output point feature class.
        points_per_meter: Point density per meter (default 0.5 => spacing 2m).

    Returns:
        Number of points created in the output.
    """
    desc = arcpy.Describe(polygon_fc)
    extent = desc.extent

    width = float(extent.XMax - extent.XMin)
    height = float(extent.YMax - extent.YMin)

    logging.info("Polygon extent size: %.2f x %.2f meters", width, height)
    logging.info("Extent area (approx): %.2f m^2", width * height)

    if width < 1.0 or height < 1.0:
        logging.info("Very small polygon: creating centroid only.")
        safe_delete(output_fc)
        arcpy.FeatureToPoint_management(polygon_fc, output_fc, "CENTROID")
        return 1

    if points_per_meter <= 0:
        raise ValueError("points_per_meter must be > 0")

    point_spacing = 1.0 / points_per_meter
    cols = max(2, int(math.ceil(width / point_spacing)))
    rows = max(2, int(math.ceil(height / point_spacing)))
    logging.info("Grid: %d x %d = %d points (spacing=%.2f m)", rows, cols, rows * cols, point_spacing)

    # Generate coordinates
    x_coords: List[float] = []
    y_coords: List[float] = []

    for col in range(cols):
        x = float(extent.XMin + col * point_spacing)
        if x <= extent.XMax:
            x_coords.append(x)

    for row in range(rows):
        y = float(extent.YMin + row * point_spacing)
        if y <= extent.YMax:
            y_coords.append(y)

    points = [arcpy.Point(x, y) for x in x_coords for y in y_coords]

    # Create temp point FC in memory
    temp_fc = arcpy.CreateFeatureclass_management(
        "in_memory",
        "grid_temp_points",
        "POINT",
        spatial_reference=desc.spatialReference,
    )[0]

    with arcpy.da.InsertCursor(temp_fc, ["SHAPE@"]) as cur:
        for p in points:
            cur.insertRow([p])

    # Select points within polygon
    points_layer = "in_memory/points_layer"
    arcpy.MakeFeatureLayer_management(temp_fc, points_layer)
    arcpy.SelectLayerByLocation_management(points_layer, "WITHIN", polygon_fc)

    count = int(arcpy.GetCount_management(points_layer)[0])
    safe_delete(output_fc)

    if count > 0:
        arcpy.CopyFeatures_management(points_layer, output_fc)
    else:
        logging.info("No points within polygon after selection: creating centroid.")
        arcpy.FeatureToPoint_management(polygon_fc, output_fc, "CENTROID")
        count = 1

    safe_delete(points_layer)
    safe_delete(temp_fc)
    return count


def create_points_using_fishnet(
    polygon_fc: str,
    output_fc: str,
    cell_size: float = 2.0,
) -> int:
    """
    Use ArcPy CreateFishnet and keep points within polygon.

    Args:
        polygon_fc: Input polygon feature class.
        output_fc: Output point feature class.
        cell_size: Fishnet cell size in meters.

    Returns:
        Number of output points.
    """
    desc = arcpy.Describe(polygon_fc)
    extent = desc.extent

    width = float(extent.XMax - extent.XMin)
    height = float(extent.YMax - extent.YMin)

    # Adjust for very small polygons
    if width < cell_size or height < cell_size:
        cell_size = max(0.1, min(width, height) / 2.0)

    rows = max(2, int(math.ceil(height / cell_size)))
    cols = max(2, int(math.ceil(width / cell_size)))

    logging.info("Creating fishnet: %dx%d (cell_size=%.3f m)", rows, cols, cell_size)

    fishnet_poly = arcpy.CreateFishnet_management(
        out_feature_class="in_memory/fishnet_poly",
        origin_coord=f"{extent.XMin} {extent.YMin}",
        y_axis_coord=f"{extent.XMin} {extent.YMin + cell_size}",
        cell_width=cell_size,
        cell_height=cell_size,
        number_rows=rows,
        number_columns=cols,
        labels="LABELS",
        geometry_type="POLYGON",
    )[0]

    # Convert fishnet polygons to label points (centroids)
    fishnet_labels = "in_memory/fishnet_labels"
    arcpy.FeatureToPoint_management(fishnet_poly, fishnet_labels, "CENTROID")

    # Select points within polygon
    points_inside = arcpy.SelectLayerByLocation_management(fishnet_labels, "WITHIN", polygon_fc)

    count = int(arcpy.GetCount_management(points_inside)[0])
    safe_delete(output_fc)

    if count > 0:
        arcpy.CopyFeatures_management(points_inside, output_fc)
    else:
        logging.info("No fishnet points inside polygon: creating centroid.")
        arcpy.FeatureToPoint_management(polygon_fc, output_fc, "CENTROID")
        count = 1

    safe_delete(points_inside)
    safe_delete(fishnet_labels)
    safe_delete(fishnet_poly)
    return count


def simple_grid_points(
    polygon_fc: str,
    output_fc: str,
    spacing: float = 0.5,
) -> int:
    """
    Simple spacing-based grid inside extent; then keep only points within polygon.

    Args:
        polygon_fc: Input polygon feature class.
        output_fc: Output point feature class.
        spacing: Grid spacing in meters.

    Returns:
        Number of output points.
    """
    if spacing <= 0:
        raise ValueError("spacing must be > 0")

    desc = arcpy.Describe(polygon_fc)
    extent = desc.extent

    x_min, y_min = float(extent.XMin), float(extent.YMin)
    x_max, y_max = float(extent.XMax), float(extent.YMax)

    # Create a temp FC for all points (will be filtered by spatial selection)
    temp_fc = arcpy.CreateFeatureclass_management(
        arcpy.env.workspace,
        "temp_grid_points__to_delete",
        "POINT",
        spatial_reference=desc.spatialReference,
    )[0]

    points: List[arcpy.Point] = []
    x = x_min
    while x <= x_max:
        y = y_min
        while y <= y_max:
            points.append(arcpy.Point(x, y))
            y += spacing
        x += spacing

    with arcpy.da.InsertCursor(temp_fc, ["SHAPE@"]) as cur:
        for p in points:
            cur.insertRow([p])

    layer = "temp_points_layer"
    arcpy.MakeFeatureLayer_management(temp_fc, layer)
    arcpy.SelectLayerByLocation_management(layer, "WITHIN", polygon_fc)

    safe_delete(output_fc)
    arcpy.CopyFeatures_management(layer, output_fc)

    count = int(arcpy.GetCount_management(output_fc)[0])

    safe_delete(layer)
    safe_delete(temp_fc)
    return count


# ================================
# Section 5: Main Runner
# ================================

def choose_method_and_generate(
    polygon_fc: str,
    cfg: Config,
) -> tuple[str, int]:
    """
    Choose a generation method based on polygon extent area and generate points.

    Returns:
        (output_fc_name, points_count)
    """
    width, height, area = polygon_metrics(polygon_fc)
    logging.info("Processing %s | size=%.1f x %.1f m | area=%.1f m^2", polygon_fc, width, height, area)

    output_fc = f"{polygon_fc}_DensePoints"
    safe_delete(output_fc)

    if area <= cfg.centroid_area_threshold:
        logging.info("Area <= %.1f: centroid-only.", cfg.centroid_area_threshold)
        arcpy.FeatureToPoint_management(polygon_fc, output_fc, "CENTROID")
        return output_fc, 1

    if area <= cfg.grid_area_threshold:
        logging.info("Area <= %.1f: simple grid (spacing=%.3f m).", cfg.grid_area_threshold, cfg.simple_grid_spacing_m)
        count = simple_grid_points(polygon_fc, output_fc, spacing=cfg.simple_grid_spacing_m)
        return output_fc, count

    # Fishnet cell size heuristic (similar to the original logic)
    cell_size = math.sqrt(area) / 5.0
    cell_size = max(cfg.fishnet_cell_min_m, min(cell_size, cfg.fishnet_cell_max_m))
    logging.info("Fishnet method (cell_size=%.3f m).", cell_size)

    count = create_points_using_fishnet(polygon_fc, output_fc, cell_size=cell_size)
    return output_fc, count


def main(cfg: Config) -> int:
    """Main entrypoint."""
    configure_logging()
    set_workspace(cfg.workspace_gdb)
    ensure_overwrite(cfg.overwrite_output)

    logging.info("Workspace: %s", cfg.workspace_gdb)
    logging.info("Polygon prefix: %s", cfg.polygon_prefix)

    targets = list_target_polygons(cfg.polygon_prefix)
    logging.info("Found %d target polygon FC(s).", len(targets))

    if not targets:
        logging.error("No target polygons found. Nothing to do.")
        return 2

    successful = 0
    for idx, poly_fc in enumerate(targets, 1):
        logging.info("(%d/%d) %s", idx, len(targets), poly_fc)
        try:
            out_fc, count = choose_method_and_generate(poly_fc, cfg)
            logging.info("Created %s (%d points).", out_fc, count)
            successful += 1
        except Exception as ex:
            logging.exception("Failed processing %s: %s", poly_fc, str(ex))
            # Fallback: centroid-only
            try:
                out_fc = f"{poly_fc}_DensePoints"
                safe_delete(out_fc)
                arcpy.FeatureToPoint_management(poly_fc, out_fc, "CENTROID")
                logging.info("Fallback centroid created: %s", out_fc)
                successful += 1
            except Exception:
                logging.exception("Fallback centroid also failed for %s.", poly_fc)

    dense_fcs = arcpy.ListFeatureClasses("*_DensePoints") or []
    logging.info("Summary: success=%d | fail=%d", successful, max(0, len(targets) - successful))

    if dense_fcs:
        logging.info("Generated DensePoints feature classes:")
        for fc in dense_fcs:
            try:
                cnt = int(arcpy.GetCount_management(fc)[0])
                logging.info(" - %s: %d points", fc, cnt)
            except Exception:
                logging.info(" - %s: count unavailable", fc)

    return 0 if successful > 0 else 1


if __name__ == "__main__":
    # Update this path to your GDB workspace
    CONFIG = Config(
        workspace_gdb=r"M:\Reza\Survey\WGIS\P24_GOSAL\Gosal\Gosal.gdb",
        polygon_prefix="o",
        overwrite_output=True,
    )
    raise SystemExit(main(CONFIG))
