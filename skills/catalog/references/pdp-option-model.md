# PDP Option Model (Reference Implementation)

Use this when implementing PDP option UI that behaves consistently across variant and non-variant products.

## Goals

- Option query params are canonical URL state (`size`, `color`, etc.).
- Variant resolution always matches all option keys against `variant.associated_options`.
- Non-variant products can still render option-style UI from `attributes` for brand consistency.
- Add to Cart is gated by valid variant resolution only for `has_variant: true`.
- Option group rendering is driven by `variant_options` order and values.

## TypeScript Reference

```typescript
import type {
  AssociatedOption,
  Product,
  ProductAttribute,
  Variant,
  VariantOption,
} from "@commercengine/storefront-sdk";

type SupportedOptionType = "color" | "single-select";
type AssociatedOptionValue = AssociatedOption[string];

type OptionDisplayValue = {
  key: string;
  label: string;
  hexcode?: string;
  isPurchasable: boolean;
};

type OptionDisplayGroup = {
  key: string;
  name: string;
  type: SupportedOptionType;
  source: "variant" | "attribute";
  selectable: boolean;
  values: OptionDisplayValue[];
};

type UrlSyncResult = {
  changed: boolean;
  nextParams: URLSearchParams;
};

const isSupportedOptionType = (type: string): type is SupportedOptionType =>
  type === "color" || type === "single-select";

const isSupportedAttribute = (
  attr: ProductAttribute
): attr is Extract<ProductAttribute, { type: SupportedOptionType }> =>
  isSupportedOptionType(attr.type);

const normalizeAssociatedOption = (option: AssociatedOptionValue) => {
  if (option.type === "color") {
    return {
      key: option.value.name,
      label: option.value.name,
      hexcode: option.value.hexcode,
    };
  }

  return { key: option.value, label: option.value };
};

const normalizeAttributeValues = (
  attr: Extract<ProductAttribute, { type: SupportedOptionType }>
): Omit<OptionDisplayValue, "isPurchasable">[] => {
  if (attr.type === "color") {
    return attr.value.map((color) => ({
      key: color.name,
      label: color.name,
      hexcode: color.hexcode,
    }));
  }

  return [{ key: attr.value, label: attr.value }];
};

const normalizeVariantOptionValues = (
  option: VariantOption & { type: SupportedOptionType }
): Omit<OptionDisplayValue, "isPurchasable">[] => {
  if (option.type === "color") {
    return option.value.flatMap((item) => {
      if (item && typeof item === "object" && "name" in item && "hexcode" in item) {
        const color = item as { name: string; hexcode: string };
        return [{ key: color.name, label: color.name, hexcode: color.hexcode }];
      }
      return [];
    });
  }

  return option.value.flatMap((item) => {
    if (typeof item === "string") return [{ key: item, label: item }];
    return [];
  });
};

const getVariantOption = (variant: Variant, optionKey: string): AssociatedOptionValue | null => {
  const option = variant.associated_options?.[optionKey];
  if (!option) return null;
  return option.type === "color" || option.type === "single-select" ? option : null;
};

const isPurchasable = (stockAvailable: boolean, backorder?: boolean) =>
  stockAvailable || Boolean(backorder);

const matchesSelection = (variant: Variant, selection: Record<string, string>) =>
  Object.entries(selection).every(([key, expected]) => {
    const option = getVariantOption(variant, key);
    if (!option) return false;
    return normalizeAssociatedOption(option).key === expected;
  });

const getOptionKeys = (product: Product) =>
  product.variant_options?.map((option) => option.key) ?? [];

const hasAllOptionsSelected = (
  optionKeys: string[],
  selectedOptions: Record<string, string>
) => optionKeys.length > 0 && optionKeys.every((key) => !!selectedOptions[key]);

const findVariantBySelection = (
  variants: Variant[],
  optionKeys: string[],
  selectedOptions: Record<string, string>
) => {
  if (!hasAllOptionsSelected(optionKeys, selectedOptions)) return null;

  return (
    variants.find((variant) => {
      return optionKeys.every((key) => {
        const option = getVariantOption(variant, key);
        return option ? normalizeAssociatedOption(option).key === selectedOptions[key] : false;
      });
    }) ?? null
  );
};

const getDefaultVariant = (product: Product) => {
  if (!product.has_variant || product.variants.length === 0) return null;
  return product.variants.find((variant) => variant.is_default) ?? product.variants[0] ?? null;
};

const getVariantOptionSelection = (variant: Variant, optionKeys: string[]) => {
  const selection: Record<string, string> = {};
  for (const key of optionKeys) {
    const option = getVariantOption(variant, key);
    if (!option) continue;
    selection[key] = normalizeAssociatedOption(option).key;
  }
  return selection;
};

const getSelectedOptionsFromSearchParams = (
  searchParams: URLSearchParams,
  optionKeys: string[]
) => {
  const selection: Record<string, string> = {};
  for (const key of optionKeys) {
    const value = searchParams.get(optionQueryParamKey(key));
    if (value) selection[key] = value;
  }
  return selection;
};

export function buildVariantOptionGroups(
  product: Product,
  selectedOptions: Record<string, string>
): OptionDisplayGroup[] {
  if (!product.has_variant || !product.variant_options) return [];

  return product.variant_options
    .filter((option): option is VariantOption & { type: SupportedOptionType } =>
      isSupportedOptionType(option.type)
    )
    .map((option) => {
      const values = normalizeVariantOptionValues(option).map((value) => {
        const candidateSelection = { ...selectedOptions, [option.key]: value.key };
        const isPurchasableForValue = product.variants.some((variant) => {
          return (
            matchesSelection(variant, candidateSelection) &&
            isPurchasable(variant.stock_available, variant.backorder)
          );
        });

        return { ...value, isPurchasable: isPurchasableForValue };
      });

      return {
        key: option.key,
        name: option.name,
        type: option.type,
        source: "variant",
        selectable: true,
        values,
      };
    });
}

export function buildAttributeOptionGroups(
  product: Product,
  brandOptionKeys: Set<string> = new Set()
): OptionDisplayGroup[] {
  if (product.has_variant) return [];

  return product.attributes
    .filter(isSupportedAttribute)
    .filter((attr) => brandOptionKeys.size === 0 || brandOptionKeys.has(attr.key))
    .map((attr) => ({
      key: attr.key,
      name: attr.name,
      type: attr.type,
      source: "attribute",
      selectable: false,
      values: normalizeAttributeValues(attr).map((value) => ({
        ...value,
        isPurchasable: isPurchasable(product.stock_available, product.backorder),
      })),
    }));
}

// Avoid duplicate rendering: hide attributes already rendered as option-style groups.
export function getRenderableAttributes(
  product: Product,
  optionGroups: OptionDisplayGroup[]
): ProductAttribute[] {
  const renderedOptionKeys = new Set(optionGroups.map((group) => group.key));
  if (renderedOptionKeys.size === 0) return product.attributes;
  return product.attributes.filter((attr) => !renderedOptionKeys.has(attr.key));
}

// URL canonical key rule
export const optionQueryParamKey = (optionKey: string) => optionKey;

// URL state sync: option params are canonical, variant is derived/backfill only.
export function syncVariantUrlState(
  product: Product,
  searchParams: URLSearchParams
): UrlSyncResult {
  if (!product.has_variant) {
    return { changed: false, nextParams: new URLSearchParams(searchParams) };
  }

  const optionKeys = getOptionKeys(product);
  const nextParams = new URLSearchParams(searchParams);
  let changed = false;

  const variantFromUrlId = nextParams.get("variant");
  const variantFromUrl = variantFromUrlId
    ? product.variants.find((variant) => variant.id === variantFromUrlId) ?? null
    : null;

  const selectedOptions = getSelectedOptionsFromSearchParams(nextParams, optionKeys);
  const selectedVariant = findVariantBySelection(product.variants, optionKeys, selectedOptions);
  const hasAnyOptionParam = optionKeys.some((key) => nextParams.has(optionQueryParamKey(key)));

  if (optionKeys.length === 0) {
    const bootstrapVariant = variantFromUrl ?? getDefaultVariant(product);
    if (bootstrapVariant && nextParams.get("variant") !== bootstrapVariant.id) {
      nextParams.set("variant", bootstrapVariant.id);
      changed = true;
    }
    return { changed, nextParams };
  }

  if (!hasAnyOptionParam) {
    const bootstrapVariant = variantFromUrl ?? getDefaultVariant(product);
    if (bootstrapVariant) {
      const defaultSelection = getVariantOptionSelection(bootstrapVariant, optionKeys);
      for (const optionKey of optionKeys) {
        const value = defaultSelection[optionKey];
        if (!value) continue;
        const queryKey = optionQueryParamKey(optionKey);
        if (nextParams.get(queryKey) !== value) {
          nextParams.set(queryKey, value);
          changed = true;
        }
      }
      if (nextParams.get("variant") !== bootstrapVariant.id) {
        nextParams.set("variant", bootstrapVariant.id);
        changed = true;
      }
    }

    return { changed, nextParams };
  }

  if (selectedVariant) {
    if (nextParams.get("variant") !== selectedVariant.id) {
      nextParams.set("variant", selectedVariant.id);
      changed = true;
    }

    return { changed, nextParams };
  }

  if (variantFromUrl) {
    const variantSelection = getVariantOptionSelection(variantFromUrl, optionKeys);
    let filledMissingOption = false;

    for (const optionKey of optionKeys) {
      const queryKey = optionQueryParamKey(optionKey);
      if (nextParams.has(queryKey)) continue;

      const value = variantSelection[optionKey];
      if (!value) continue;

      nextParams.set(queryKey, value);
      changed = true;
      filledMissingOption = true;
    }

    if (!filledMissingOption && nextParams.has("variant")) {
      nextParams.delete("variant");
      changed = true;
    }

    return { changed, nextParams };
  }

  if (nextParams.has("variant")) {
    nextParams.delete("variant");
    changed = true;
  }

  return { changed, nextParams };
}
```

## URL State Rules

- Canonical: option params are source of truth.
- Derived: `variant` query param is optional convenience.
- Backfill: if URL has valid `variant` but missing options, fill only missing option params from that variant.
- Cleanup: if options are invalid/incomplete and no backfill is possible, remove `variant`.
