<template>
    <input
        ref="searchElementRef"
        name="select-search"
        :style="[searchStyles]"
        :class="['ww-select-search']"
        @input="handleInputChange"
        @focus="handleSearchFocus"
        @blur="handleSearchBlur"
        :placeholder="searchPlaceholder"
    />
</template>

<script>
import { inject, onMounted, onBeforeUnmount, ref, computed, watch, nextTick } from 'vue';
/* wwEditor:start */
import useEditorHint from './editor/useEditorHint';
/* wwEditor:end */
import { debounce } from './utils';

export default {
    props: {
        content: { type: Object, required: true },
        /* wwEditor:start */
        wwEditorState: { type: Object, required: true },
        /* wwEditor:end */
        wwElementState: { type: Object, required: true },
    },
    emits: ['element-event', 'trigger-event', 'update:sidepanel-content'],
    setup(props, { emit }) {
        /* wwEditor:start */
        useEditorHint(emit);
        /* wwEditor:end */
        const { updateHasSearch, updateSearchElement, updateSearch, autoFocusSearch, focusSearch, isSearchBarFocused } = inject(
            '_wwSelect:useSearch',
            {}
        );
        const rawData = inject('_wwSelect:rawData', ref([]));
        const searchState = inject('_wwSelect:searchState', ref(null));
        
        const searchElementRef = ref(null);
        const searchElement = computed(() => searchElementRef.value);
        const searchBy = computed(() => {
            return (props.content.searchBy || [])
                .filter(item => item && item.filter)
                .map(item => JSON.parse(item.filter.replace(/'/g, '"')))
                .flat();
        });
        
        const options = computed(() => {
            const items = rawData.value;
            return Array.isArray(items) ? items : [];
        });

        // Helper function to filter options based on search value
        const filterOptions = (options, filterValue) => {
            if (!filterValue) return options;
            
            return options.filter(option => {
                // Handle primitive values directly
                const isPrimitive = typeof option !== 'object' || option === null;
                if (isPrimitive) {
                    const normalizedOption = option
                        .toString()
                        .normalize('NFD')
                        .replace(/[\u0300-\u036f]/g, '')
                        .toLowerCase();
                    const normalizedFilter = filterValue
                        .normalize('NFD')
                        .replace(/[\u0300-\u036f]/g, '')
                        .toLowerCase();
                    return normalizedOption.includes(normalizedFilter);
                } else {
                    // For objects, use the existing search logic
                    const searchByFields = searchState.value?.searchBy?.length
                        ? searchState.value?.searchBy
                        : Object.keys(option);
                    return searchByFields.some(key => {
                        const optionValue = option[key];
                        if (!optionValue) return false;
                        const normalizedOption = optionValue
                            .toString()
                            .normalize('NFD')
                            .replace(/[\u0300-\u036f]/g, '')
                            .toLowerCase();
                        const normalizedFilter = filterValue
                            .normalize('NFD')
                            .replace(/[\u0300-\u036f]/g, '')
                            .toLowerCase();

                        return normalizedOption.includes(normalizedFilter);
                    });
                }
            });
        };

        const debouncedUpdateSearch = debounce((value, searchBy) => {
            console.log('[Search] debouncedUpdateSearch called:', { value, searchBy });
            if (updateSearch) {
                console.log('[Search] Calling updateSearch');
                // Compute searchMatches here in the Search component
                const searchMatches = value ? filterOptions(options.value, value) : [];
                updateSearch({ value, searchBy, searchMatches });
                console.log('[Search] updateSearch complete, matches:', searchMatches.length);
            }
        }, 300);

        const searchStyles = computed(() => {
            const borderCss = !props.content.searchBorder
                ? {
                      border: props.content.searchBorderAll,
                  }
                : {
                      'border-top': props.content.searchBorderTop,
                      'border-right': props.content.searchBorderRight,
                      'border-bottom': props.content.searchBorderBottom,
                      'border-left': props.content.searchBorderLeft,
                  };

            return {
                width: props.content.searchWidth,
                height: props.content.searchHeight,
                'border-radius': props.content.searchBorderRadius,
                padding: props.content.searchPadding,
                margin: props.content.searchMargin,
                'background-color': props.content.searchBgColor,
                'font-family': props.content.searchFontFamily,
                'font-size': props.content.searchFontSize,
                'font-weight': props.content.searchFontWeight,
                color: props.content.searchFontColor,
                '--placeholder-color': props.content.searchPlaceholderColor,
                outline: props.content.searchOutline,
                'outline-offset': props.content.searchOutlineOffset,
                cursor: 'text',
                ...borderCss,
            };
        });

        const searchPlaceholder = computed(() => {
            return wwLib.wwLang.getText(props.content.searchPlaceholder);
        });

        // This event come from ww-input-basic => https://github.com/weweb-assets/ww-input-basic
        const handleInputChange = event => {
            console.log('[Search] handleInputChange called:', event?.target?.value);
            debouncedUpdateSearch(event?.target?.value, searchBy);
            console.log('[Search] handleInputChange - debounced update queued');
        };

        const handleSearchFocus = () => {
            isSearchBarFocused.value = true;
        };

        const handleSearchBlur = () => {
            isSearchBarFocused.value = false;
        };

        watch(searchElement, value => {
            if (updateSearchElement) updateSearchElement(value);
        });

        onMounted(() => {
            console.log('[Search] Component mounted');
            if (updateHasSearch) updateHasSearch(true);
            if (updateSearch) {
                console.log('[Search] Initializing search with empty value');
                updateSearch({ value: '', searchBy, searchMatches: [] });
            }
        });

        onBeforeUnmount(() => {
            console.log('[Search] Component unmounting');
            if (updateHasSearch) updateHasSearch(false);
        });

        return {
            searchElementRef,
            handleInputChange,
            handleSearchFocus,
            handleSearchBlur,
            searchStyles,
            searchPlaceholder,
        };
    },
};
</script>

<style scoped lang="scss">
.ww-select-search {
    &::placeholder {
        color: var(--placeholder-color);
    }
}
</style>
